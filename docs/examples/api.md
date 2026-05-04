---
title: API
sidebar: example
---
# API Examples

Below are examples on how to use the library independently with the API.

## Log in - Username and Password with MFA (If Required)

Below is an example how to log in to Hive with 2FA if needed
and get a session token.

```Python
import pyhiveapi as Hive

tokens = {}
hive_auth = Hive.Auth(username="<Hive Username>", password="<Hive Password>")
auth_data = hive_auth.login()

if auth_data.get("ChallengeName") == Hive.SMS_REQUIRED:
    code = input("Enter your 2FA code: ")
    auth_data = hive_auth.sms_2fa(code, auth_data)
    hive_auth.device_registration('MyPyHiveAPIDevice')
    print(f"device_group_key: {hive_auth.device_group_key}")
    print(f"device_key: {hive_auth.device_key}")
    print(f"device_password: {hive_auth.device_password}")

if "AuthenticationResult" in auth_data:
    session = auth_data["AuthenticationResult"]
    tokens.update({"token": session["IdToken"]})
    tokens.update({"refreshToken": session["RefreshToken"]})
    tokens.update({"accessToken": session["AccessToken"]})

```


## Log in - Using Device Authentication

Below is an example how to log in to Hive with 2FA if needed
and get a session token.

```Python
import pyhiveapi as Hive

tokens = {}
hive_auth = Hive.Auth(<Hive Username>", "<Hive Password>", "Hive Device Group Key>", "<Hive Device Key>", "<Hive Device Password>")
auth_data = hive_auth.device_login()

if "AuthenticationResult" in auth_data:
    auth_result = session["AuthenticationResult"]
    tokens.update({"token": session["IdToken"]})
    tokens.update({"refreshToken": session["RefreshToken"]})
    tokens.update({"accessToken": session["AccessToken"]})
```


## Refresh Tokens

Below is an example how to refresh your session tokens
after they have expired

```Python
import pyhiveapi as Hive

tokens = {}
hive_auth = Hive.Auth(<Hive Username>", "<Hive Password>", "Hive Device Group Key>", "<Hive Device Key>", "<Hive Device Password>")
auth_data = hive_auth.device_login()
new_tokens = hive_auth.refresh_token(tokens['AuthenticationResult']['RefreshToken'])

if "AuthenticationResult" in new_tokens:
    session = new_tokens["AuthenticationResult"]
    tokens.update({"token": session["IdToken"]})
    tokens.update({"refreshToken": session["RefreshToken"]})
    tokens.update({"accessToken": session["AccessToken"]})
```

## Get Hive Data - Using Tokens

Below is an example how to data from the Hive platform
using the session token acquired from login.

```Python
api = Hive.API(token=tokens["token"])
data = api.getAll()
```
