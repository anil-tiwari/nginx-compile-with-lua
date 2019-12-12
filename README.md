# Building Nginx 1.17 from source with LuaJIT 2.0!

Module included

    resty core
    lrucache module
    redis module

**Nginx is a great webserver**. But it has no scripting capabilities. To add scripting capabilities in Nginx, one needs to build it from source with the necessary add-ons.
In this article I will showcase how we can build **Nginx with Lua**.

## OpenResty
It is highly recommended to use OpenResty releases which bundle Nginx, ngx_lua (this module), LuaJIT, as well as other powerful companion Nginx modules and Lua libraries.

It is discouraged to build this module with Nginx yourself since it is tricky to set up exactly right.

Note that Nginx, LuaJIT, and OpenSSL official releases have various limitations and long standing bugs that can cause some of this module's features to be disabled, not work properly, or run slower. 
Official OpenResty releases are recommended because they bundle [OpenResty's optimized LuaJIT 2.1](https://github.com/openresty/luajit2) fork and [Nginx/OpenSSL patches](https://github.com/openresty/openresty/tree/master/patches).

#### Alternatively, ngx_lua can be manually compiled into Nginx:
                
1. LuaJIT can be downloaded from the [latest release of OpenResty's LuaJIT fork](https://github.com/openresty/luajit2/releases). The official LuaJIT 2.x releases are also supported, although performance will be significantly lower for reasons elaborated above
2. Download the latest version of the ngx_devel_kit (NDK) module [HERE](https://github.com/simplresty/ngx_devel_kit/tags)
3. Download the latest version of ngx_lua [HERE](https://github.com/openresty/lua-nginx-module/tags)
4. Download the latest supported version of Nginx [HERE](HERE) (See [Nginx Compatibility](https://github.com/openresty/lua-nginx-module#nginx-compatibility))
                
#### Build the source with this module:
    -- Download compatible Nginx 
        wget http://nginx.org/download/nginx-1.17.6.tar.gz
    -- Download LuaJit2
        wget -O luajit2_2.1.tar.gz https://github.com/openresty/luajit2/archive/v2.1-20190912.tar.gz
    -- Download Nginx Devel Kit
        wget -O nginx_devel_kit.tar.gz https://github.com/simplresty/ngx_devel_kit/archive/v0.3.1.tar.gz
    -- Download nginx lua module
        wget -O nginx_lua_module.tar.gz https://github.com/openresty/lua-nginx-module/archive/v0.10.15.tar.gz
    -- Download resty-core
        wget -O lua-resty-core.tar.gz https://github.com/openresty/lua-resty-core/archive/v0.1.17.tar.gz
    -- Download resty-lrucache
        wget -O lua-resty-lrucache.tar.gz https://github.com/openresty/lua-resty-lrucache/archive/v0.09.tar.gz
    -- Download resty-redis
        wget -O lua-resty-redis.tar.gz https://github.com/openresty/lua-resty-redis/archive/v0.27.tar.gz

Extract all the tar files

    tar xvf nginx-1.17.6.tar.gz
    tar xvf luajit2_2.1.tar.gz
    tar xvf nginx_devel_kit.tar.gz
    tar xvf nginx_lua_module.tar.gz
    tar xvf lua-resty-core.tar.gz
    tar xvf lua-resty-lrucache.tar.gz
    tar xvf lua-resty-redis.tar.gz

### Building LuaJIT
To build Nginx with LuaJIT, we need to build LuaJIT first. This is as simple as a make command

    cd luajit2_2.1
    make install 

### Building Nginx
Next we build configure the Nginx build. We also need to set the environment variables to specifcy the location of the LuaJIT library

    LUAJIT_LIB=/usr/local/lib LUAJIT_INC=/usr/local/include/luajit-2.1 \
    ./configure \
    --user=nobody                          \
    --group=nobody                         \
    --prefix=/etc/nginx                    \
    --sbin-path=/usr/sbin/nginx            \
    --conf-path=/etc/nginx/nginx.conf      \
    --pid-path=/var/run/nginx.pid          \
    --lock-path=/var/run/nginx.lock        \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --with-ld-opt="-Wl,-rpath,/path/to/luajit/lib" \
    --with-http_gzip_static_module         \
    --with-http_stub_status_module         \
    --with-http_ssl_module                 \
    --with-pcre                            \
    --with-file-aio                        \
    --with-http_realip_module              \
    --without-http_scgi_module             \
    --without-http_uwsgi_module            \
    --without-http_fastcgi_module ${NGINX_DEBUG:+--debug} \
    --with-cc-opt=-O2 --with-ld-opt='-Wl,-rpath,/usr/local/lib' \
    --add-module=/path/to/devel/ngx_devel_kit-0.3.1 \
    --add-module=/path/to/nginx-lua/lua-nginx-module-0.10.15
**make install**    
**Run nginx command to test config**
    
    nginx -t
    nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    nginx: configuration file /etc/nginx/nginx.conf test is successful

**Note** : you might get error while running "nginx -t" so add user nobody and nogroup in nginx.conf

Open **/etc/nginx/nginx.conf** and add below line in it
    user  nobody nogroup;
    

**Install below module before starting nginx**
    
    cd lua-resty-core
    make install

    cd lua-resty-lrucache
    make install

    cd lua-resty-redis
    make install

**Now you can run nginx**
    
    nginx

Now let put the Nginx Lua to a test.
### Nginx Lua Testing
We will build a simple lua endpoint http://localhost/sum?a=<num>&b=<num>, which sums two numbers passed through the url get parameters
    
    location /sum {
      content_by_lua_block {
        local args = ngx.req.get_uri_args();
        ngx.say(args.a + args.b)
      }
    }

Insert the above lines of code in the \etc\nginx\nginx.conf inside the server block. Then start nginx in background  
    
    $ nginx &
    $ curl "http://localhost/sum/?a=10&b=20"
    30 
 
Now let put the Nginx redis to a test.
### Nginx redis Testing
Here I just added a key in redis with name adsadsaw334342121 having some value "Welcome to redis-lua module".

We will build a simple lua endpoint http://localhost/test/?key=adsadsaw334342121

    location /test {
        content_by_lua_block {
            local args = ngx.req.get_uri_args();
            ngx.say(args.key )
            local redis = require "resty.redis"
            local red = redis:new()

            red:set_timeouts(1000, 1000, 1000) -- 1 sec

            -- or connect to a unix domain socket file listened
            -- by a redis server:
            --     local ok, err = red:connect("unix:/path/to/redis.sock")

            local ok, err = red:connect("127.0.0.1", 6370)
            ngx.header.content_type = 'application/json';
            if not ok then
                ngx.say("failed to connect: ", err)
                return
            end

            local res, err = red:get(args.key)
            if not res then
                ngx.say("failed to get dog: ", err)
                return
            end

            if res == ngx.null then
                ngx.say("dog not found.")
                return
            end

            ngx.say(args.key, res)

            -- put it into the connection pool of size 100,
            -- with 10 seconds max idle time
            local ok, err = red:set_keepalive(10000, 100)
            if not ok then
                ngx.say("failed to set keepalive: ", err)
                return
            end

            -- or just close the connection right away:
            -- local ok, err = red:close()
            -- if not ok then
            --     ngx.say("failed to close: ", err)
            --     return
            -- end
        }
    }

Insert the above lines of code in the \etc\nginx\nginx.conf inside the server block. Then re-start nginx in background  
    
    $ nginx &
    $ curl "http://localhost/sum/?key=adsadsaw334342121"
    Welcome to redis-lua module 

 
@Refrence From [HERE](https://github.com/openresty/lua-nginx-module) 
