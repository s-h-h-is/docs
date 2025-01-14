---
id: api
title: API
tags:
- proof_service
sidebar_position: 10
---

## General

### Entrypoint {#entrypoint}

| Online | Environment | Entrypoint                        | Healthcheck                               |
|--------|-------------|-----------------------------------|-------------------------------------------|
| ✅     | Staging     | https://proof-service.nextnext.id | https://proof-service.nextnext.id/healthz |
| ✅     | Production  | https://proof-service.next.id     | https://proof-service.next.id/healthz     |

### Structure

All requests and responses should be `Content-Type: application/json`.

### Supported platforms for proofing

See [Platform supported](ps-platforms-supported)

### Post struct placeholders

| placeholders   | Should be replaced to    | Sample                                                                                     |
|----------------|--------------------------|--------------------------------------------------------------------------------------------|
| `%SIG_BASE64%` | Base64-encoded signature | `1uZDzxZ6wae+IaF4BgJXWAWC9e/nxbkdC0xp+xRLz1FqeghynyW+SQnGQygdgQYLTLfXqq03nyFQJU0GidQ/3QA=` |

## Group Common

### General info [GET /healthz] {#healthz}

+ Request (application/json)

+ Response 200 (application/json)

  + Attributes (object)

    + hello (string, required) - must be `proof server`.
    + platforms (array[string], required) - All `platform`s supported by this server.

  + Body

        {
          "hello": "proof server",
          "platforms": [
              "github",
              "twitter",
              "ethereum",
              "keybase"
          ]
        }

## Group Proof

### Query a proof payload to signature and to post [POST /v1/proof/payload] {#proof-payload}

+ Request (application/json)

  + Attributes (object)

    + action (string, required) - Action (`create` / `delete`)
    + platform (string, required) - Target platform. See table above for all available platforms. See table in [Platform supported](ps-platforms-supported) for all available values.
    + identity (string, required) - Identity in target platform to proof. Usually a "username" or "screen name". See [Platform supported](ps-platforms-supported).
    + public_key (string, required) - Public key of Persona to connect to. Should be secp256k1 curve (for now), 65-bytes or 33-bytes long (uncompressed / compressed) and stringified into hex form (`/^0x[0-9a-f]{65,130}$/`).

  + Body

        {
          "action": "create",
          "platform": "twitter",
          "identity": "my_twitter_screen_name",
          "public_key": "0x04c7cacde73af939c35d527b34e0556ea84bab27e6c0ed7c6c59be70f6d2db59c206b23529977117dc8a5d61fa848f94950422b79d1c142bcf623862e49f9e6575"
        }

+ Response 200 (application/json)

  + Attributes (object)

    + post_content (object, required) - Post (in different languages) to let user send / save to target platform.
        Placeholders should be replaced by frontend / client.
        Language code follows BCP-47 standard (i.e. https://docs.microsoft.com/en-us/openspecs/office_standards/ms-oe376/6c085406-a698-4e12-9d4d-c3b0ee3dbc4a ).
        Note: there is always a `default` content.
    + sign_payload (string, required) - Raw string to be sent to `personal_sign`
    + uuid (string, required) - UUID of this chain link. Send this UUID to `POST /v1/proof` as-is.
    + created_at (string, required) - Creation time of this chain link (UNIX timestamp, unit: second). Send this to `POST /v1/proof` as-is.

  + Body

        {
          "post_content": {
            "default": "Prove myself: I'm 0x028c3cda474361179d653c41a62f6bbb07265d535121e19aedf660da2924d0b1e3 on Next.ID. Signature: %SIG_BASE64%"
            "zh_CN": "验证推特账号 @my_twitter_screen_name 的 Next.ID 身份 @NextDotID 。\nSig: %SIG_BASE64%\n\n请下载安装 mask.io 去使用您 Web 3.0 的去中心化身份。\n"
          },
          "sign_payload": "{\"action\":\"add\",\"identity\":\"my_twitter_screen_name\",\"platform\":\"twitter\",\"prev\":null}"
          "uuid": "ed9f421d-92e1-4c80-9bff-8516ef46ff43",
          "created_at": "1647332405"
        }

### Add a proof modification into Proof chain [POST /v1/proof] {#proof-add}

+ Request (application/json)

  + Attributes (object)

    + action (string, required) - Action (`create` / `delete`)
    + platform (string, required) - Target platform. See table above for all available platforms. See table above for all available values.
    + identity (string, required) - Identity in target platform to proof. Usually a "username" or "screen name". See [Platform supported](ps-platforms-supported).
    + proof_location (string, optional) - Location where public-accessible proof post is set. See [Platform supported](ps-platforms-supported).
    + public_key (string, required) - Public key of Next.ID Persona to connect to. Should be secp256k1 curve (for now), 65-bytes or 33-bytes long (uncompressed / compressed) and stringified into hex form (`/^0x[0-9a-f]{65,130}$/`).
    + extra (object, optional) - Extra info for specific platform needed. See [Flow](ps-flow#ethereum) for more info.
      + wallet_signature (string, optional) - (required when `platform: ethereum`) Signature signed by ETH wallet (w/ same sign payload), BASE64-ed.
      + signature (string, optional) - (required when `platform: ethereum` or `action: delete`) Signature signed by Persona private key (w/ same sign payload), BASE64-ed.
    + uuid (string, required) - UUID of this chain link. Use the exact value from `POST /v1/proof/payload`.
    + created_at (string, required) - Creation time of this chain link (UNIX timestamp, unit: second). Use the exact value from `POST /v1/proof/payload`.

  + Body

        {
          "action": "create",
          "platform": "twitter",
          "identity": "my_twitter_screen_name",
          "proof_location": "1415362679095635970",
          "public_key": "0x04c7cacde73af939c35d527b34e0556ea84bab27e6c0ed7c6c59be70f6d2db59c206b23529977117dc8a5d61fa848f94950422b79d1c142bcf623862e49f9e6575",
          "extra": {},
          "uuid": "ed9f421d-92e1-4c80-9bff-8516ef46ff43",
          "created_at": "1647332405"
        }

+ Response 201 (application/json)

Request submitted successfully. No error happened.

+ Response 400 (application/json)

Request failed.

    + Attributes

      + message (string, required) - Contains some error info for user.

    + Body

        {
           "message": "Tweet author is not the same as given identity."
        }

### Query an existed binding [GET /v1/proof] {#proof-query}

+ Request

    + Parameters

      + platform (string, optional) - Proof platform. If not given, all platforms will be searched.
      + identity (string, required) - Identity on target platform. Separate identities with comma (`,`) if you want to query mutipe identity at once.
      + page (number, optional) - Pagination. First page is number `1`.

    + Example

      `GET /proof?platform=twitter&identity=my_twitter_screen_name`

      `GET /proof?identity=abc,def&page=3`

+ Response 200 (application/json)

  + Attributes

    + pagination (object, required) - Pagination info
      + total (number, required) - Total amount of results.
      + per (number, required) - How many `ids` results per page.
      + current (number, required) - current page number.
      + next (number, required) - Next page. `0` if current page is the last one.
    + ids (array[object], required) - All IDs found. Will be empty array if not found.
      + persona (string, required) - Persona public key
      + proofs (array[object], required) - All proofs under this persona
        + platform (string, required) - Platform
        + identity (string, required) - Identity on that platform
        + created_at (string, required) - Creation time of this proof. (timestamp, unit: second)
        + last_checked_at (string, required) - When last validation happened. (timestamp, unit: second)
        + is_valid (bool, required) - This record is valid or not according to last validation.
        + invalid_reason (string, required) - If not valid, reason will appears here.

  + Body

        {
          "pagination": {
            "total": 27,
            "per": 10,
            "current": 1,
            "next": 2
          },
          "ids": [{
            "persona": "0x04c7cacde73af939c35d527b34e0556ea84bab27e6c0ed7c6c59be70f6d2db59c206b23529977117dc8a5d61fa848f94950422b79d1c142bcf623862e49f9e6575",
            "proofs": [{
              "platform": "twitter",
              "identity": "my_twitter_screen_name",
              "created_at": "1643099438",
              "last_checked_at": "1643099438",
              "is_valid": false,
              "invalid_reason": "tweet not found"
            }, {
              "platform": "facebook",
              "identity": "my_facebook_username",
              "created_at": "1643099438",
              "last_checked_at": "1643099438",
              "is_valid": true,
              "invalid_reason": ""
            }]
          }, {
            "persona": "0xANOTHER",
            "proofs": [{
              "platform": "ethereum",
              "identity": "0x114514......",
              "created_at": "1643099438",
              "last_checked_at": "1643099438",
              "is_valid": true,
              "invalid_reason": ""
            }]
          }]
        }

### Check if a proof exists [GET /v1/proof/exists] {#proof-query-exists}

+ Request

  + Parameters

    + platform (string, required) - Proof platform.
    + identity (string, required) - Identity on target platform.
    + public_key (string, required) - Public key of Next.ID Persona to connect to. Should be secp256k1 curve (for now), 65-bytes or 33-bytes long (uncompressed / compressed) and stringified into hex form (`/^0x[0-9a-f]{65,130}$/`)

  + Example

    `GET /v1/proof/exists?platform=twitter&identity=some_twitter_screenname&public_key=0x04c7cacde73af939c35d527b34e0556ea84bab27e6c0ed7c6c59be70f6d2db59c206b23529977117dc8a5d61fa848f94950422b79d1c142bcf623862e49f9e6575`

+ Response 200 (application/json)

Found.

  + Attributes

    + created_at (string, required) - Createtion time of this proof. (timestamp, unit: second)
    + last_checked_at (string, required) - When last validation happened. (timestamp, unit: second)
    + is_valid (bool, required) - This record is valid or not according to last validation.
    + invalid_reason (string, required) - If not valid, reason will appears here.

  + Body

        {
          "created_at": "1643099438",
          "last_checked_at": "1643099438",
          "is_valid": true,
          "invalid_reason": ""
        }

+ Response 404 (application/json)

Not found.

  + Attributes

    + message (string, required) - Message of which part goes wrong.
