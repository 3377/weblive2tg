>[!TIP]
>一款基于GITHUB ACTION定时自动测活，保活的工具<br>
>场景用于koyeb保活，huggingface等需要一定时间内有活动的项目保活

# 效果图
![image](https://github.com/user-attachments/assets/0b01e978-9b81-4cab-ac1f-b1b5a9619299)

# 新建一个仓库
公开和私有都可以，尽量私有，此方法私有变量写到了页面中
点开，action，在workflow下新建main.yml，内容如下
## 自动检测代码
```yaml
name: 网站状态定时检查

on:
  workflow_dispatch:  # 保留手动触发选项
  schedule:
    - cron: '0 */6 * * *'  # 每6小时运行一次

env:
    WEBSITES: |
      [
        "https://XXX",
        "https://XXX",
        "https://XXX"
      ]
    TG_BOT_TOKEN: 'XXX' #替换为您j机器人实际 bot token
    TG_CHAT_ID: 'XXX' #替换为您的实际 chat ID

jobs:
  check-websites:
    runs-on: ubuntu-latest
    steps:
    - name: 检查网站状态
      id: check
      run: |
        websites=$(echo $WEBSITES | jq -r '.[]')
        current_utc_time=$(date "+%Y-%m-%d %H:%M:%S")
        current_china_time=$(date -d "$current_utc_time UTC+8 hours" "+%Y-%m-%d   %H:%M:%S")
        report="<b>📊 网站状态报告</b>\n\n<b>📅 检查时间:</b> ${current_china_time}\n\n"
        
        for website in $websites; do
          status_code=$(curl -o /dev/null -s -w "%{http_code}" $website)
          if [ $status_code -eq 200 ]; then
            status="✅ 正常"
          else
            status="❌ 异常"
          fi
          result="🌐 <a href='${website}'>${website}</a>\n   状态: ${status} (状态码: ${status_code})\n\n"
          echo -e "$result"
          report="${report}${result}"
        done
        
        echo "report<<EOF" >> $GITHUB_OUTPUT
        echo -e "$report" >> $GITHUB_OUTPUT
        echo "EOF" >> $GITHUB_OUTPUT

    - name: 发送报告到 Telegram
      run: |
        report="${{ steps.check.outputs.report }}"
        curl -s -X POST "https://api.telegram.org/bot${TG_BOT_TOKEN}/sendMessage" \
          -H "Content-Type: application/json" \
          -d "{\"chat_id\": \"${TG_CHAT_ID}\", \"text\": \"${report}\", \"parse_mode\": \"HTML\", \"disable_web_page_preview\": true}"

    - name: 显示报告
      run: |
        echo -e "${{ steps.check.outputs.report }}"
