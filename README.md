# Marzneshin SHM Billing template

Шаблон для связки SHM Billing с Marzneshin

## Use

#### В settings сервера добавить

```
marzneshin {
    link: "https://Marzneshin.link"
    username: "username"
    password: "password"
    services: "1,2,3"
}
```
**services - id services из Marzneshin**
<img width="904" alt="Снимок экрана 2024-10-16 в 15 36 28" src="https://github.com/user-attachments/assets/3acf0aa0-4943-482a-b7d5-dbffea089d2a">

#### Добавить шаблон Marzneshin
```
#!/bin/bash

EVENT="{{ event_name }}"
SESSION_ID="{{ user.gen_session.id }}"
API_URL="{{ config.api.url }}"
MARZNESHIN_HOST="{{ server.settings.marzneshin.link }}"
MARZNESHIN_USERNAME="{{server.settings.marzneshin.username}}"
MARZNESHIN_PASSWORD="{{server.settings.marzneshin.password}}"


echo "Marzneshin template for SHM"
echo
echo "EVENT=$EVENT"

get_marzneshin_token() {
    if [ -f "/etc/opt/marzneshin/.env" ]; then

        echo "Marzneshin host: $MARZNESHIN_HOST"

        export TOKEN=$(curl -sk -XPOST \
          "$MARZNESHIN_HOST/api/admins/token" \
          -H 'Content-Type: application/x-www-form-urlencoded' \
          -d "grant_type=password&username=$MARZNESHIN_USERNAME&password=$MARZNESHIN_PASSWORD" | jq -r .access_token)

        if [ -z "$TOKEN" ]; then
            echo 'Error: can not get TOKEN. Please check docker containers status'
            exit 1
        fi
    else
        echo 'Error: Marzneshin is not installed! Please check settings'
        exit 1
    fi
}

case $EVENT in
    INIT)

        echo "Check SHM API host: $API_URL"
        echo "Marzneshin host: $MARZNESHIN_HOST"
        echo "Marzneshin user: $MARZNESHIN_USERNAME"
        echo "Marzn pass: $MARZNESHIN_PASSWORD"
        HTTP_CODE=$(curl -sk -o /dev/null -w "%{http_code}" $API_URL/shm/v1/test)
        RET_CODE=$?
        if [ $RET_CODE -ne 0 ]; then
            echo "Error: host $API_URL is incorrect."
            echo "Please set correct public host in SHM config. It must be accessible from the server."
            exit 1
        fi
        if [ $HTTP_CODE -ne '200' ]; then
            echo "ERROR: incorrect API URL: $API_URL"
            echo "Got status: $HTTP_CODE"
            exit 1
        fi
        ;;
    CREATE)
        echo "Create a new user"

        PAYLOAD="$(cat <<-EOF
        {
          "username": "us_{{ us.id }}",
          "data_limit": 0,
          "expire_date": null,
          "expire_strategy": "never",
          "service_ids": [{{ server.settings.marzneshin.services }}],
          "note": "SHM_info- {{ user.login }}, {{ user.full_name }}, https://t.me/{{ user.settings.telegram.login }}"
        }
EOF
        )"

        get_marzneshin_token
        USER_CFG=$(curl -sk -XPOST \
          "$MARZNESHIN_HOST/api/users" \
          -H "Authorization: Bearer $TOKEN" \
          -H 'Content-Type: application/json' \
          -d "$PAYLOAD")

        if [ -z $(echo "$USER_CFG" | jq -r '.username | select( . != null )') ]; then
            echo "Error: $USER_CFG"
            exit 1
        fi

        echo "Upload user config to SHM: $API_URL/shm/v1/storage/manage/vpn_mrzn_{{ us.id }}"
        curl -sk -XPUT \
            -H "session-id: $SESSION_ID" \
            -H "Content-Type: application/json" \
            $API_URL/shm/v1/storage/manage/vpn_mrzn_{{ us.id }} \
            --data-binary "$USER_CFG"
        echo "done"
        ;;
    ACTIVATE)
        echo "Activate user"

        get_marzneshin_token
        USER_CFG=$(curl -sk -XPOST \
          "$MARZNESHIN_HOST/api/users/us_{{ us.id }}/enable" \
          -H "Authorization: Bearer $TOKEN" \
          -H 'Content-Type: application/json' \
          -d "")

        if [ -z $(echo "$USER_CFG" | jq -r '.username | select( . != null )') ]; then
            echo "Error: $USER_CFG"
            exit 1
        fi

        echo "Update user config to SHM: $API_URL/shm/v1/storage/manage/vpn_mrzn_{{ us.id }}"
        curl -sk -XPOST \
            -H "session-id: $SESSION_ID" \
            -H "Content-Type: application/json" \
            $API_URL/shm/v1/storage/manage/vpn_mrzn_{{ us.id }} \
            --data-binary "$USER_CFG"

        echo "done"
        ;;
    BLOCK)
        echo "Block user"

        get_marzneshin_token
        USER_CFG=$(curl -sk -XPOST \
          "$MARZNESHIN_HOST/api/users/us_{{ us.id }}/disable" \
          -H "Authorization: Bearer $TOKEN" \
          -H 'Content-Type: application/json' \
          -d "")

        if [ -z $(echo "$USER_CFG" | jq -r '.username | select( . != null )') ]; then
            echo "Error: $USER_CFG"
            exit 1
        fi

        echo "Update user config to SHM: $API_URL/shm/v1/storage/manage/vpn_mrzn_{{ us.id }}"
        curl -sk -XPOST \
            -H "session-id: $SESSION_ID" \
            -H "Content-Type: application/json" \
            $API_URL/shm/v1/storage/manage/vpn_mrzn_{{ us.id }} \
            --data-binary "$USER_CFG"

        echo "done"
        ;;
    REMOVE)
        echo "Remove user"

        get_marzneshin_token
        curl -sk -XDELETE \
          "$MARZNESHIN_HOST/api/users/us_{{ us.id }}" \
          -H "Authorization: Bearer $TOKEN"

        echo "Remove user key from SHM"
        curl -sk -XDELETE \
            -H "session-id: $SESSION_ID" \
            $API_URL/shm/v1/storage/manage/vpn_mrzn_{{ us.id }}
        echo "done"
        ;;
    PROLONGATE)
        echo "Reset user counters"

        get_marzneshin_token
        USER_CFG=$(curl -sk -XPOST \
          "$MARZNESHIN_HOST/api/users/us_{{ us.id }}/reset" \
          -H "Authorization: Bearer $TOKEN")

        if [ -z $(echo "$USER_CFG" | jq -r '.username | select( . != null )') ]; then
            echo "Error: $USER_CFG"
            exit 1
        fi

        echo "Update user config to SHM: $API_URL/shm/v1/storage/manage/vpn_mrzn_{{ us.id }}"
        curl -sk -XPOST \
            -H "session-id: $SESSION_ID" \
            -H "Content-Type: application/json" \
            $API_URL/shm/v1/storage/manage/vpn_mrzn_{{ us.id }} \
            --data-binary "$USER_CFG"

        echo "done"
        ;;
    *)
        echo "Unknown event: $EVENT. Exit."
        exit 0
        ;;
esac

```

