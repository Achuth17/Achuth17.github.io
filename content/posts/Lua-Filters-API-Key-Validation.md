+++
title = 'Envoy Proxy: API Key Validation using HTTP Lua Filters'
date = 2024-10-20T11:07:45-07:00
draft = false
tags = ['envoy', 'lua']
+++

As a continuation of my previous [blog post](https://achuth.blog/posts/envoy-lua-filter-exploration/), I was looking for good examples to further explore Envoy's extension points. During this search, I found an [issue](https://github.com/envoyproxy/envoy/issues/34877) on Github asking for a native Envoy Filter to do API Key Validation. I am not very familiar with Envoy's native C++ filters but, as things stand today, we can already solve this problem using a Lua HTTP Filter. This post will explain how.

## What are API Keys?

[API Keys](https://swagger.io/docs/specification/v3_0/authentication/api-keys/) are used for authorization by some servers and they are usually a short unique string used to identify client. Although, API Keys are not the most secure form of authorization, they are still commonly used. When a client makes an API call to the server, this unique API Key can be included in the request in the following ways:

### Header

```
curl  -H "X-API-KEY: PERMITTED_API_KEY" 'localhost:10000/get'

GET /get HTTP/1.1
Host: localhost:10000
User-Agent: curl/8.7.1
Accept: */*
X-API-KEY: PERMITTED_API_KEY
```

### Query Parameter

```
curl -vvv 'localhost:10000/get?X-API-KEY=PERMITTED_API_KEY'

GET /get?X-API-KEY=PERMITTED_API_KEY HTTP/1.1
Host: localhost:10000
User-Agent: curl/8.7.1
Accept: */*
```

### Cookie

```
curl -vvv --cookie "X-API-KEY=PERMITTED_API_KEY" 'localhost:10000/get'

GET /get HTTP/1.1
Host: localhost:10000
User-Agent: curl/8.7.1
Accept: */*
Cookie: X-API-KEY=PERMITTED_API_KEY
```

## How are API Keys validated in Envoy?

Since Envoy doesn't natively support API Key Validation, there are a few ways to extend Envoy to perform API Key Validation:

1. [External Authorization Filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_authz_filter)
2. [External Processing Filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_proc_filter)
3. [WASM Filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/wasm_filter)
4. [Lua Filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/lua_filter)

In the post, I will be exploring the Lua HTTP Filter based approach.

## API Key Validation using HTTP Lua Filter

As I discussed in the previous [blog post](https://achuth.blog/posts/envoy-lua-filter-exploration/), it is possible to extract or modify headers, query parameters, body and trailers of a request or response using a custom Lua script. This script can also be selectively attach to some or all paths in an Envoy proxy. Building on this, I will be making the following three changes in this new API Key Validation Lua script -

1. Extract the API Key from Headers or Query Params.
2. Validate the API Key by making an external HTTP Call.
3. Terminate requests that fail API Key Validation step with a proper error message.

[Full example](https://github.com/Achuth17/envoy-configs/blob/main/lua-filter/envoy-lua-apikey-auth.yaml)

### Extracting API Key from Headers

```
local auth_key = "X-API-KEY"
local headers = request_handle:headers()
api_key_value = headers:get(auth_key)
```

### Extracting API Key from Query Params

```
local auth_key = "X-API-KEY"
local path = request_handle:headers():get(":path")
local api_key_param_value = extract_query_param_from_path(path, auth_key)

function extract_query_param_from_path(path, param_name)
  start_index = string.find(path, "?")
  if start_index == nil then
    return
  end
  q_param_string = string.sub(path, start_index+1)
  q_param_pairs = split(q_param_string, "&")
  for i,param_pair in pairs(q_param_pairs) do
    param_split = split(param_pair, "=")
    if (param_split[1] == param_name) then
      return param_split[2]
    end
  end
end

-- Source: https://stackoverflow.com/questions/1426954/split-string-in-lua
function split(s, separator)
  local fields = {}
  
  local insert_func = function (val)
    table.insert(fields, val)
  end

  local pattern = string.format("([^%s]+)", separator)
  string.gsub(s, pattern, insert_func)

  return fields
end
```

### Making an external validation call

We can make an HTTP call from the request or respose flow in the Lua Filter. In this case, I have opted to use the sync HTTP call. The backend server used for making the validation call should be [registered](https://github.com/Achuth17/envoy-configs/blob/main/lua-filter/envoy-lua-apikey-auth.yaml#L53) as a `cluster` in the main envoy config.yaml file.

```
local resp_headers, body = request_handle:httpCall(
    "auth_cluster",
    {
    [":method"] = "POST",
    [":path"] = "/auth",
    [":authority"] = "auth_cluster",
    ["X-API-KEY"] = api_key_value
    },
    "",
    5000)
status_code = resp_headers[":status"]
```

### Rejecting requests that fail validation

If the API Key Validation Server does not respond back with a 200 status code, we can reject the request made by the client with a canned "Request Denied" response and a 403 status code.

```
if resp_headers[status_header] ~= "200" then
    -- Respond back as denied, we can short circuit any further filters.
    request_handle:respond({[":status"] = status_code}, "API Key is Invalid!\n")
end
```

[Full example](https://github.com/Achuth17/envoy-configs/blob/main/lua-filter/envoy-lua-apikey-auth.yaml)

To validate the API Keys, I am hosting a [simple go http server](https://github.com/Achuth17/envoy-configs/blob/main/lua-filter/auth_server/auth_server.go) that rejects requests that don't contain the expected API Key in the header.

## Learnings:

1. The Lua HTTP Filter is fairly extensible as it can make one or more sync/async HTTP calls in the request or response path.
2. The Lua HTTP Filter can terminate any request or response early by sending back a canned response.