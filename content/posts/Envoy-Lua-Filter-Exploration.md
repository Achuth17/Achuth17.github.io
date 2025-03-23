+++
title = 'Envoy Lua HTTP Filter Exploration'
date = 2024-10-13T00:11:52-07:00
draft = false
+++

Envoy offers a of variety in HTTP filters and you can do a lot with what comes out of the box like `Header Mutation filter` or `gRPC-HTTP Transcoding`. While this is great, there are certain circumstances where we need some additional customizations. To solve this issue, Envoy provides options like Lua HTTP Filter, Wasm Filter or the ExtProc Filter. Through these filters, you can write custom code in a non-C++ language that gets excuted either in the request or response flow like any other Envoy Filter. To start my journey into extending Envoy, I spent some time looking into the Lua HTTP Filter.

## Writing and maintaining Lua Scripts

### In-lining Lua Scripts
Register the script as an `inline_string` in the `default_source_code` or in the `source_codes` map section of the Lua HTTP Filter.

```
...
http_filters:
- name: envoy.filters.http.lua
typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua              
    default_source_code:
    inline_string: |
        -- Called on the request path.
        function envoy_on_request(request_handle)
        request_handle:logInfo("Logging in the request path from default lua source.")
        end
        -- Called on the response path.
        function envoy_on_response(response_handle)
        response_handle:logInfo("Logging in the response path from default lua source.")
        end
...
```

### Lua Script Files
Create a separate Lua script file, Implement the two stub methods function `envoy_on_request(request_handle)` and function `envoy_on_response(response_handle)`. Ensure the script is accessible from the Envoy runtime. Once this is done, the script file can be referenced from the `source_codes` map section of the Lua HTTP Filter.

```
-- file.lua
-- Called on the request path.
function envoy_on_request(request_handle)
    local headers = request_handle:headers()
    header_value_to_replace = "User-Agent"
    header_value = headers:get(header_value_to_replace)
    if header_value ~= nil then
      request_handle:logInfo("Replacing UserAgent header found in the request with 'Envoy'")
      request_handle:headers():remove(header_value_to_replace)
      request_handle:headers():add(header_value_to_replace, "Envoy")
    end
  end

  -- Called on the response path.
  function envoy_on_response(response_handle)
  end
```
Register the script file in the `source_codes` map in the Lua HTTP Filter.

```
...
http_filters:
          - name: envoy.filters.http.lua
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua              
              source_codes:
                file.lua:
                  filename: file.lua
...
```

## Attaching Lua Scripts

### Lua Per Route
In each applicable route defined in a `virtual_host`,  the `source_codes` map entry for the lua script should be referenced in a `typed_per_filter_config`.

```
...
filter_chains:
    - filters:     
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/ip" }
                route: { cluster: some_service }
                typed_per_filter_config:
                  envoy.filters.http.lua:
                    '@type': "type.googleapis.com/envoy.extensions.filters.http.lua.v3.LuaPerRoute"
                    name: "file.lua"
...
```

### Default Lua Script
Default scripts are added by default to all routes except in routes with `Lua Per Route`. No extra steps are required in the `virtual_host` or `route` configurations.


## Learnings:

1. Default Lua Script gets executed only when `Lua Per Route` is not in play. If `Lua Per Route` is used, the Default Lua Script is ignored.

2. Scenarios where Lua HTTP Filter is useful in its current form:

    i. Modifying Headers, Body and Trailers including cases where the body is chunked or streamed.

    ii. Making an external HTTP call to an upstream cluster. This can be done to either to log the request or response, send stats to an external server or perform auth. The suggestion is to not make blocking calls in the Lua HTTP Filter.

    iii. Short circuiting request or response flows. If the headers, body or trailers don’t meet a certain criteria, we can short circuit the request or response flow and send back a canned response to the client. In the request flow, This could prevent calls from reaching the upstream cluster. 
