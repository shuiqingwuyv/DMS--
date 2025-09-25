# DMS修改登录页面欢迎信息和标题(我的系统是7.2 其他版本自测)
## 1.计划任务
设置-计划任务-计划的任务-用户自定义的脚本
名字随便取账户root
![image](https://github.com/shuiqingwuyv/DMS--/blob/main/1.png)
## 2.更新时间
我设置的1小时一次
![image](https://github.com/shuiqingwuyv/DMS--/blob/main/2.png)
## 3.脚本
在任务设置中粘贴以下脚本 然后确定 在手动运行一次
```
#!/bin/sh
# keai_update_synoinfo.sh
# 说明：从 https://keai.icu/apiwyy/api 获取 user/music/content，
# 并将 /etc/synoinfo.conf 中的 login_welcome_title/login_welcome_msg 两行替换为：
#   login_welcome_title="MUSIC_VAL"
#   login_welcome_msg="CONTENT_VAL——USER_VAL"
#
# 要求：以 root 身份运行以便写 /etc/synoinfo.conf

set -eu

URL="https://keai.icu/apiwyy/api"
SYNOCONF="/etc/synoinfo.conf"
TIMESTAMP="$(date +%Y%m%d%H%M%S)"
BACKUP="${SYNOCONF}.bak.${TIMESTAMP}"
TMPFILE="$(mktemp /tmp/synoinfo.conf.XXXXXX)" || exit 1

# --------------- 获取 JSON 并解析为变量 ---------------
resp="$(curl -sSf "$URL")" || {
  echo "ERROR: 无法从 $URL 获取内容"
  exit 2
}
# remove CR if any
resp="$(printf '%s' "$resp" | tr -d '\r')"

# parse json to USER_VAL, MUSIC_VAL, CONTENT_VAL
if command -v jq >/dev/null 2>&1; then
  USER_VAL="$(printf '%s' "$resp" | jq -r '.user // ""')"
  MUSIC_VAL="$(printf '%s' "$resp" | jq -r '.music // ""')"
  CONTENT_VAL="$(printf '%s' "$resp" | jq -r '.content // ""')"
elif command -v python3 >/dev/null 2>&1 || command -v python >/dev/null 2>&1; then
  PY_BIN="$(command -v python3 || command -v python)"
  USER_VAL="$(printf '%s' "$resp" | $PY_BIN -c "import sys,json; j=json.load(sys.stdin); print(j.get('user',''))")"
  MUSIC_VAL="$(printf '%s' "$resp" | $PY_BIN -c "import sys,json; j=json.load(sys.stdin); print(j.get('music',''))")"
  CONTENT_VAL="$(printf '%s' "$resp" | $PY_BIN -c "import sys,json; j=json.load(sys.stdin); print(j.get('content',''))")"
else
  # 简单 sed 回退（仅适用于无复杂转义的简单 JSON）
  USER_VAL="$(printf '%s' "$resp" | sed -n 's/.*"user"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p')"
  MUSIC_VAL="$(printf '%s' "$resp" | sed -n 's/.*"music"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p')"
  CONTENT_VAL="$(printf '%s' "$resp" | sed -n 's/.*"content"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p')"
fi

# --------------- 准备替换值（在双引号内安全写入） ---------------
# 将值中的反斜杠、双引号、美元符号转义，确保写进 synoinfo.conf 的双引号字符串不会断裂
escape_for_double_quotes() {
  # 参数：原始字符串
  printf '%s' "$1" \
    | sed -e 's/\\/\\\\/g' -e 's/"/\\"/g' -e 's/\$/\\$/g'
}

ESC_MUSIC="$(escape_for_double_quotes "$MUSIC_VAL")"
ESC_CONTENT="$(escape_for_double_quotes "$CONTENT_VAL")"
ESC_USER="$(escape_for_double_quotes "$USER_VAL")"

NEW_TITLE="login_welcome_title=\"$ESC_MUSIC\""
NEW_MSG="login_welcome_msg=\"$ESC_CONTENT --$ESC_USER\""

# --------------- 检查文件并备份 ---------------
if [ ! -f "$SYNOCONF" ]; then
  echo "ERROR: $SYNOCONF 不存在，无法替换。"
  exit 3
fi

# 确保有写权限（常需 root）
if [ ! -w "$SYNOCONF" ]; then
  echo "ERROR: 没有权限写 $SYNOCONF，请使用 root 或 sudo 运行脚本。"
  exit 4
fi

cp -p "$SYNOCONF" "$BACKUP" || {
  echo "ERROR: 备份失败 $BACKUP"
  exit 5
}
echo "已备份原文件到 $BACKUP"

# --------------- 逐行处理并替换（保留其他内容不变） ---------------
# 如果原文件中没有对应变量行，则会在末尾追加这两行（保持结构）
title_replaced=0
msg_replaced=0

# 使用 awk 做行处理并写入临时文件（原子更新）
awk -v new_title="$NEW_TITLE" -v new_msg="$NEW_MSG" '
BEGIN {
  title_replaced=0; msg_replaced=0;
}
/^[[:space:]]*login_welcome_title[[:space:]]*=.*/ {
  print new_title;
  title_replaced=1;
  next;
}
/^[[:space:]]*login_welcome_msg[[:space:]]*=.*/ {
  print new_msg;
  msg_replaced=1;
  next;
}
{ print }
END {
  # 如果某一项未被替换，追加到文件末尾
  if (title_replaced==0) print new_title;
  if (msg_replaced==0) print new_msg;
}
' "$SYNOCONF" > "$TMPFILE" || {
  echo "ERROR: 写入临时文件失败"
  exit 6
}

# --------------- 原子替换并输出结果 ---------------
# 将临时文件移动覆盖原文件（保留文件权限）
# 为保持权限，先保存权限然后覆盖并恢复权限
orig_perms=$(stat -c %a "$SYNOCONF" 2>/dev/null || stat -f %Mp%Lp "$SYNOCONF" 2>/dev/null || echo "0644")
orig_owner=$(stat -c %u:%g "$SYNOCONF" 2>/dev/null || stat -f "%u:%g" "$SYNOCONF" 2>/dev/null || echo "0:0")

# Move tmp to target (use mv to preserve dir semantics)
mv "$TMPFILE" "$SYNOCONF" || {
  echo "ERROR: 无法移动临时文件覆盖 $SYNOCONF"
  rm -f "$TMPFILE"
  exit 7
}

# 恢复权限与所有者（尝试）
chmod "$orig_perms" "$SYNOCONF" 2>/dev/null || true
chown "$orig_owner" "$SYNOCONF" 2>/dev/null || true

echo "替换完成："
echo "  $NEW_TITLE"
echo "  $NEW_MSG"
echo "原文件已备份到： $BACKUP"
echo "注意：若 synology 在 web 管理页面缓存欢迎信息，可能需要重启相应服务或重启设备以生效。"

exit 0

```
