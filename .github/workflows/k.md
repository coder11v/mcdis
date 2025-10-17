name: Check Minecraft Server Status

on:
  schedule:
    # Runs every 5 minutes
    - cron: '*/5 * * * *'
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

jobs:
  check-server:
    runs-on: ubuntu-latest
    steps:
      # 1. Check out your repo so we can read/write the status file
      - name: Checkout repo
        uses: actions/checkout@v4

      # 2. Read the last known status from a file
      - name: Read last status
        id: last_status
        run: |
          if [ -f .server-status ]; then
            echo "status=$(cat .server-status)" >> $GITHUB_OUTPUT
          else
            echo "status=offline" >> $GITHUB_OUTPUT
          fi

      # 3. Check the current server status from the API
      - name: Check current server status
        id: current_status
        run: |
          STATUS_JSON=$(curl -s "https://api.mcsrvstat.us/2/YOURSERVER.aternos.me")
          IS_ONLINE=$(echo $STATUS_JSON | jq -r '.online')
          if [ "$IS_ONLINE" = "true" ]; then
            echo "status=online" >> $GITHUB_OUTPUT
          else
            echo "status=offline" >> $GITHUB_OUTPUT
          fi

      # 4. Send webhook ONLY if status changed from offline to online
      - name: Send Discord notification
        if: steps.last_status.outputs.status == 'offline' && steps.current_status.outputs.status == 'online'
        env:
          DISCORD_WEBHOOK: ${{ secrets.DISCORD_WEBHOOK }}
        run: |
          echo "Server came ONLINE! Sending notification."
          curl -H "Content-Type: application/json" \
               -d '{"username": "Minecraft Monitor", "embeds": [{"title": "🟢 Server Online!", "description": "Your Aternos server is now online!", "color": 65280}]}' \
               $DISCORD_WEBHOOK

      # 5. Update and commit the status file IF it has changed
      - name: Update status file
        if: steps.last_status.outputs.status != steps.current_status.outputs.status
        run: |
          echo ${{ steps.current_status.outputs.status }} > .server-status
          git config --global user.name 'GitHub Actions Bot'
          git config --global user.email 'github-actions-bot@users.noreply.github.com'
          git add .server-status
          git commit -m "Update server status to ${{ steps.current_status.outputs.status }}"
          git push
