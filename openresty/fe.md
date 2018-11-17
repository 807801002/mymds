# nginx.conf

```nginx
#user  nobody;
worker_processes 15;
worker_cpu_affinity 000000000000001 000000000000010 000000000000100 000000000001000 000000000010000 000000000100000 000000001000000 000000010000000 000000100000000 000001000000000 000010000000000 000100000000000 001000000000000 010000000000000 100000000000000;
worker_rlimit_nofile 65535;

#error_log  logs/error.log info;
error_log  logs/error.log error;

events {
    use epoll;
    worker_connections  65535;
}
http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$upstream_addr" $request_time $upstream_response_time';
    access_log  logs/access.log  main;

    sendfile       on;
    tcp_nopush     on;
    keepalive_timeout  60;
    msie_padding off;

    lua_package_path "/usr/local/openresty/nginx/conf/lua/?.lua;;";  #lua 模块

    lua_shared_dict pods  2m;
    lua_shared_dict rules 2m;
    lua_shared_dict rule_items 5m;
    lua_shared_dict locks 1m;
    lua_shared_dict temps 10m;

    init_worker_by_lua_file conf/lua/init.lua;
    lua_code_cache on;

    upstream backend {
        server 0.0.0.1:8000;   # just an invalid address as a place holder

        balancer_by_lua_file conf/lua/balance.lua;
    }
    
    server {
        listen 80 default_server;
        listen 443 ssl;
        listen 8443 ssl;        
        server_name _;
        charset utf-8;
        
        ssl_certificate      star_chinanetcenter_com.crt;#4_cnc.com.crt; #公钥文件名，要先将公钥文件放在 /usr/local/nginx/conf目录下
        ssl_certificate_key  star_chinanetcenter_com.key;#2_chinanetcenter.com.key.ws;
        ssl_session_cache    shared:SSL:10m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers  'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
        ssl_prefer_server_ciphers  on;
        
        location / {
            return 403;
        }
    }

    server {
        listen 80;
        listen 443 ssl;
        listen 8443 ssl;
        server_name open.chinanetcenter.com;
        keepalive_timeout 60;
        charset utf-8;        
        
        ssl_certificate      star_chinanetcenter_com.crt;#4_cnc.com.crt; #公钥文件名，要先将公钥文件放在 /usr/local/nginx/conf目录下
        ssl_certificate_key  star_chinanetcenter_com.key;#2_chinanetcenter.com.key.ws;
        ssl_session_cache    shared:SSL:10m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers  'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
        ssl_prefer_server_ciphers  on;

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
                
        location / {
            rewrite_by_lua_file conf/lua/setctx.lua;

            proxy_pass http://backend/;
            proxy_next_upstream error timeout invalid_header;
            proxy_next_upstream_tries 5;
            
            proxy_redirect off;
            proxy_set_header        Host $host;                 # don't modify!!!
            proxy_set_header        X-Remote-Addr $remote_addr; # don't modify!!!
            proxy_set_header        X-Request-Protocol http;    # don't modify!!!
            proxy_set_header        Cdn-Src-Ip $remote_addr;
            proxy_set_header        X-Real-IP $remote_addr;
            proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header        Via "nginx";
            proxy_set_header        X-Forwarded-Proto http; #注意看这里 多了一行
            proxy_connect_timeout   30s;
            proxy_send_timeout      600s;
            proxy_read_timeout      600s;
            proxy_buffers           4 32k;
            client_max_body_size    1024m;
            client_body_buffer_size 128k;
            proxy_buffering off;
            
        }
        
        location /test {
            default_type 'text/plain';
            access_by_lua_file conf/lua/access.lua;
            content_by_lua_block {
                ngx.say("success")
            }
        }

        location /refresh_pods {
            default_type 'text/plain';
            access_by_lua_file conf/lua/access.lua;
            content_by_lua_block  {
                local pods = require "pods"
                local pod = ngx.var.arg_pod
                local res = pods.refresh(pod)
                ngx.say(res)
            }
        }

        location /delete_pods {
            default_type 'text/plain';
            access_by_lua_file conf/lua/access.lua;
            content_by_lua_block  {
                local pods = require "pods"
                local pod = ngx.var.arg_pod
                local res = pods.delete(pod)
                ngx.say(res)
            }
        }
        
        location /get_pods {
            default_type 'text/plain';
            access_by_lua_file conf/lua/access.lua;
            content_by_lua_block  {
                local pods = require "pods"
                local pod = ngx.var.arg_pod
                pods.printCache(pod)
            }	
        }

        location /refresh_rules {
            default_type 'text/plain';
            access_by_lua_file conf/lua/access.lua;
            content_by_lua_block  {
                local rules = require "rules"
                local id = ngx.var.arg_id
                local res = rules.refresh(id)
                ngx.say(res)
            }
        }

        location /delete_rules {
            default_type 'text/plain';
            access_by_lua_file conf/lua/access.lua;
            content_by_lua_block  {
                local rules = require "rules"
                local id = ngx.var.arg_id
                local res = rules.delete(id)
                ngx.say(res)
            }
        }
        
        location /get_rules {
            default_type 'text/plain';
            access_by_lua_file conf/lua/access.lua;
            content_by_lua_block  {
                local rules = require "rules"
                local id = ngx.var.arg_id
                rules.printCache(id)
            }	
        }
        
        location /test_rules {
            default_type 'text/plain';
            access_by_lua_file conf/lua/access.lua;
            content_by_lua_block  {
                local rule_engine = require "rule_engine"
                local rules_ngx_cache = ngx.shared.rules
                local rule_items_ngx_cache = ngx.shared.rule_items
                local pods_ngx_cache = ngx.shared.pods

                local option = {}
                option.ip = ngx.var.arg_ip
                option.account = ngx.var.arg_account
                option.uri = ngx.var.arg_uri
                local res = rule_engine.get_upstreams(option, rules_ngx_cache, rule_items_ngx_cache, pods_ngx_cache)
                ngx.say("upstream: "..table.concat(res, ","))
            }	
        }
        
        location /get_temps {
            default_type 'text/plain';
            access_by_lua_file conf/lua/access.lua;
            content_by_lua_block  {
                local cache = ngx.shared.temps;
                local keys = cache:get_keys()

                for key,value in ipairs(keys) do
                    local val, err = cache:get(value)
                    if val then
                        ngx.say(value, ": ", val, "\r\n")
                    else
                        ngx.say("cache empty!")
                    end
                end
            }	
        }
        
        location /flush_temps {
            default_type 'text/plain';
            access_by_lua_file conf/lua/access.lua;
            content_by_lua_block  {
                local cache = ngx.shared.temps;
                cache:flush_all()
                ngx.say("success")
            }	
        }

    }

}
```

# init.lua

```lua
local start_timer = function ()
		local cache_timer = require "cache_timer"
		return cache_timer.start_once()
end

ngx.log(ngx.INFO, "before init........");

local resty_lock = require "resty.lock"

local temps = ngx.shared.temps
local val, err = temps:get("init_status")
if val then
    ngx.log(ngx.INFO, "init_status: ", val)
    return
end
if err then
		return fail("failed to get key from temps: ", err)
end

-- lock
local lock, err = resty_lock:new("locks")
if not lock then
    return fail("failed to create lock: ", err)
end

local elapsed, err = lock:lock("init_cache")
if not elapsed then
    return fail("failed to acquire the lock: ", err)
end

ngx.log(ngx.INFO, "set lock for timer");

val, err = temps:get("init_status")
if val then
		local ok, err = lock:unlock()
		if not ok then
    		return fail("failed to unlock: ", err)
    end
		ngx.log(ngx.INFO, "init_status: ", val)
    return
end

local ok, err = temps:set("init_status", "initing", 5)
if not ok then
		local ok, err = lock:unlock()
		if not ok then
    		return fail("failed to unlock: ", err)
    end

    return fail("failed to update temps cache: ", err)
end

-- start the timer to get remote config
local res = start_timer()
if not res then
    local ok, err = lock:unlock()
    if not ok then
        return fail("failed to unlock: ", err)
    end

    ngx.log(ngx.ERR, "timer start failed!!")
    return
end

local ok, err = lock:unlock()
if not ok then
		return fail("failed to unlock: ", err)
end

ngx.log(ngx.INFO, "init timer start success!!!!");

```

# cache_timer.lua

```lua
cache_timer = {}

local function refresh_all_cache()
		-- refresh pods cache
		local pods = require "pods"
		pods.refresh()
		
		-- refresh roles cache
		local rules = require "rules"
		rules.refresh()
end

function cache_timer.start_once()
		local delay = 3
		
		local handler = function (premature)
    		if not premature then
        		refresh_all_cache()
    		end
    end
		
		local ok, err = ngx.timer.at(delay, handler)
 		if not ok then
    		ngx.log(ngx.ERR, "failed to create the timer: ", err)
   			return
		end		
		return "success"
end

return cache_timer
```

# pods.lua

```lua
pods = {}

function pods.getFromRemote(pod)
		local mysql = require "resty.mysql"
		local db, err = mysql:new()
		if not db then
				ngx.log(ngx.ERR, "failed to instantiate mysql: ", err)
				return
		end
		
		db:set_timeout(1000) -- 1 sec
		
		local mysql_config = require "mysql_config"
		ngx.log(ngx.INFO, "connecting to mysql: ", mysql_config.host, ":", mysql_config.port)
		local ok, err, errcode, sqlstate = db:connect(mysql_config)
		
		if not ok then
				ngx.log(ngx.ERR, "failed to connect: ", err, ": ", errcode, " ", sqlstate)
				return
		end
		
		ngx.log(ngx.INFO, "connected to mysql.")
		
		local sql = "select * from pod"
		if pod then
				sql = sql.." where name = '"..pod.."'"
		end
		ngx.log(ngx.INFO, "sql: ", sql)
		local res, err, errcode, sqlstate = db:query(sql)
		if not res then
			ngx.log(ngx.ERR, "bad result: ", err, ": ", errcode, ": ", sqlstate, ".")
			return
		end
		
		local result = {};
		for key,value in ipairs(res) 
		do
				result[value.name] = value.ip
		end
		
		local cjson = require "cjson"
		ngx.log(ngx.INFO, "pods: ", cjson.encode(result))
		
		-- close the connection right away:
		local ok, err = db:close()
		if not ok then
				ngx.log(ngx.ERR, "failed to close: ", err)
				return
		end
		
		return result
end

function pods.refresh(pod)
		local cache = ngx.shared.pods
		local pods = pods.getFromRemote(pod)
		
		if not pods then
				return "success"
		end
		
		for key,value in pairs(pods) 
		do
				local ok, err = cache:set(key, value)
				if not ok then
						ngx.log(ngx.ERR, "failed to update cache for pods: ", err)
						ngx.log(ngx.ERR, "pods error: ", key)
						return "error"
				end
		end		

		return "success"
end

function pods.delete(pod)
		local cache = ngx.shared.pods
		
		if pod and pod ~= "" and pod ~= "default" then
				local ok, err = cache:delete(pod)
				if not ok then
						ngx.log(ngx.ERR, "fail to delete pod: ", pod)
						return "error"
				end
		end	
		return "success"
end

function pods.printCache(pod)
		local cache = ngx.shared.pods

		if pod then 
				local val, err = cache:get(pod)
				if val then
						ngx.say(pod, ": ", val, "\r\n")
				else
						ngx.say("cache empty!")
				end				
		else
				local keys = cache:get_keys()
				
				for key,value in ipairs(keys) 
				do
						local val, err = cache:get(value)
						if val then
						    ngx.say(value, ": ", val, "\r\n")
						else
								ngx.say("cache empty!")
						end
				end		
		end
		

end

function pods.isReady()
		local cache = ngx.shared.pods
		local keys = cache:get_keys()
		if #keys == 0 then
				return false				
		end
		return true
end

return pods
```

# rules.lua

```lua
rules = {}

local function getAliasList(regulation)
		local str = string.gsub(regulation,"and",",")
		str = string.gsub(str,"or",",")
		str = string.gsub(str,"not",",")
		str = string.gsub(str," ","")
		str = string.gsub(str,"%(","")
		str = string.gsub(str,"%)","")

		local string_util = require "string_util"
		local arr = string_util.split(str, ",")
		local res = {}
		for key,value in ipairs(arr) do
				if value and value ~= "" then
						table.insert(res, value)
				end
		end
		return table.concat(res, ",")
end

function rules.getFromRemote(id)
		local mysql = require "resty.mysql"
		local db, err = mysql:new()
		if not db then
				ngx.log(ngx.ERR, "failed to instantiate mysql: ", err)
				return
		end
		
		db:set_timeout(1000) -- 1 sec
		
		local mysql_config = require "mysql_config"
		ngx.log(ngx.INFO, "connecting to mysql: ", mysql_config.host, ":", mysql_config.port)
		local ok, err, errcode, sqlstate = db:connect(mysql_config)
		
		if not ok then
				ngx.log(ngx.ERR, "failed to connect: ", err, ": ", errcode, " ", sqlstate)
				return
		end
		
		ngx.log(ngx.INFO, "connected to mysql.")
		
		local sql = "select * from rule where status = 1"
		if id then
				sql = sql.." and id = "..id
		end
		ngx.log(ngx.INFO, "sql: ", sql)
		local res, err, errcode, sqlstate = db:query(sql)
		if not res then
			ngx.log(ngx.ERR, "bad result: ", err, ": ", errcode, ": ", sqlstate, ".")
			return
		end
		
		local all_rules = {};
		local rule_ids = {};
		for key,value in ipairs(res) 
		do
				all_rules[value.id] = {value.name, value.pod, value.status, value.regulation}
				local alias_list = getAliasList(value.regulation)
				table.insert(all_rules[value.id], alias_list);
				table.insert(rule_ids, value.id)
		end
		
		local cjson = require "cjson"
		cjson.encode_sparse_array(true)
		ngx.log(ngx.INFO, "rules: ", cjson.encode(all_rules))
		
		if #rule_ids == 0 then
		
				-- close the connection right away:
				local ok, err = db:close()
				if not ok then
						ngx.log(ngx.ERR, "failed to close: ", err)
						return
				end
		
				return all_rules, {}
		end
		
		local rule_ids_str = table.concat(rule_ids, ",")
		
		sql = "select * from rule_item where rule_id in ( " .. rule_ids_str .. " )"
		ngx.log(ngx.INFO, "sql: ", sql)
		res, err, errcode, sqlstate = db:query(sql)
		if not res then
			ngx.log(ngx.ERR, "bad result: ", err, ": ", errcode, ": ", sqlstate, ".")
			return
		end
		
		local rule_items = {};
		for key,value in ipairs(res) 
		do
				rule_items[value.rule_id.."_"..value.alias] = {value.type, value.pattern, value.operator}
		end
		ngx.log(ngx.INFO, "rule_items: ", cjson.encode(rule_items))
		
		-- close the connection right away:
		local ok, err = db:close()
		if not ok then
				ngx.log(ngx.ERR, "failed to close: ", err)
				return
		end
		
		return all_rules, rule_items
end

function rules.refresh(id)
		local rules, items = rules.getFromRemote(id)
		local cache = ngx.shared.rules
		
		if not rules then
				return "success"
		end
		
		for key,value in pairs(rules) 
		do
				local ok, err = cache:set(key, table.concat(value, ";"))
				if not ok then
				    ngx.log(ngx.ERR, "failed to update cache for rules: ", err)
				    ngx.log(ngx.ERR, "rules error: ", key)
				    return "error"
				end
		end
		
		cache = ngx.shared.rule_items
		for key,value in pairs(items) 
		do
				local ok, err = cache:set(key, table.concat(value, ";"))
				if not ok then
				    ngx.log(ngx.ERR, "failed to update cache for rule_items: ", err)
				    ngx.log(ngx.ERR, "rules error: ", key)
				    return "error"
				end
		end
		
		return "success"
end

function rules.delete(id)
		local cache = ngx.shared.rules
		local item_cache = ngx.shared.rule_items
		
		if id then
				local ok, err = cache:delete(id)
				if not ok then
						ngx.log(ngx.ERR, "fail to delete rule: ", id)
						return "error"
				end
				
				local keys = item_cache:get_keys()
				for key,value in ipairs(keys) 
				do
						local string_util = require "string_util"
						local arr = string_util.split(value, "_")
								
						if id == arr[1] then
								ok, err = item_cache:delete(value)
								if not ok then
										ngx.log(ngx.ERR, "fail to delete rule item: ", value)
										return "error"
								end
						end		
				end
		else
				cache:flush_all()
				item_cache:flush_all()
		end
		
		return "success"
end

function rules.printCache(id)

		ngx.say("-----rules---------")
		local cache = ngx.shared.rules;
		
		if id then
				local val, err = cache:get(id)
				if val then
						ngx.say(id, ": ", val, "\r\n")
				else
						ngx.say("cache empty!")
				end
		else
				local keys = cache:get_keys()
				
				for key,value in ipairs(keys) 
				do
						local val, err = cache:get(value)
						if val then
						    ngx.say(value, ": ", val, "\r\n")
						else
								ngx.say("cache empty!")
						end
				end		
		end
	
		ngx.say("-----rule_items---------")
		cache = ngx.shared.rule_items;
		keys = cache:get_keys()
		for key,value in ipairs(keys) 
		do
				if id then
						local string_util = require "string_util"
						local arr = string_util.split(value, "_")
						
						if id == arr[1] then
								local val, err = cache:get(value)
								if val then
								    ngx.say(value, ": ", val, "\r\n")
								else
										ngx.say("cache empty!")
								end
						end
				else
						local val, err = cache:get(value)
						if val then
						    ngx.say(value, ": ", val, "\r\n")
						else
								ngx.say("cache empty!")
						end
				end
				
				
		end
		
end

function rules.isReady()
		local cache = ngx.shared.rules;
		local keys = cache:get_keys()
		if #keys == 0 then
				return false				
		end
		return true
end

return rules
```

# rule_engine.lua

```lua
rule_engine = {}

local function match_item(option, item)
		local matcher = require "matcher"
		return matcher.match(option[item.type], item.operator, item.pattern)
end

local function match_rule(option, rule)	
		ngx.log(ngx.INFO, "regulation: ", rule.regulation)
		local final_regulation = rule.regulation
		for key, value in pairs(rule.items) do
				local res = match_item(option, value)
				if res then
						final_regulation = string.gsub(final_regulation, value.alias, "true")  -- 替换"true"，到别名
				else
						final_regulation = string.gsub(final_regulation, value.alias, "false") -- 替换"false" 到别名
				end
							
		end
		local script = "local temp = "..final_regulation..";return temp"
		local res = assert(loadstring(script))()
		ngx.log(ngx.INFO, "final_regulation: ", final_regulation, " = ", res)
		if res then
				return rule.pod, #rule.items
		end
		return "default", 0
end
-- option: ip+uri+account 
-- rules_ngx_cache: ngx.shared.rules
-- rule_items_ngx_cache: ngx.shared.rule_items
-- ngx.shared.pods
function rule_engine.get_upstreams(option, rules_ngx_cache, rule_items_ngx_cache, pods_ngx_cache)
		local cjson = require "cjson"
		cjson.encode_sparse_array(true)
		ngx.log(ngx.INFO, "option: ", cjson.encode(option))

		local string_util = require "string_util"

		local default_group = "default"
		local val, err = pods_ngx_cache:get(default_group)
		local default_upstreams = string_util.split(val, ";")

		local rule_keys = rules_ngx_cache:get_keys()
		if rule_keys == nil or #rule_keys == 0
		then
				return default_upstreams
		end
		
		local rules = {}
		local keys = rules_ngx_cache:get_keys()
		for key,value in ipairs(keys) do
				local val, err = rules_ngx_cache:get(value)
				local rule_arr = string_util.split(val, ";")
				local rule = {}
				rule.name = rule_arr[1]
				rule.pod = rule_arr[2]
				rule.status = rule_arr[3]
				rule.regulation = rule_arr[4]
				
				
				local alias_names = string_util.split(rule_arr[5], ",")
				rule.items = {}
				for k,v in pairs(alias_names) do
						local item_str = rule_items_ngx_cache:get(value.."_"..v)
						local item_arr = string_util.split(item_str, ";")
						local item = {}
						item.alias = v
						item.type = item_arr[1]
						item.pattern = item_arr[2]
						item.operator = item_arr[3]
						
						table.insert(rule.items, item)
				end
				
				
				rules[tonumber(value)] = rule
		end
				
		ngx.log(ngx.INFO, "rules: ", cjson.encode(rules))
		
		local final_pod = {}
		final_pod.name = "default"
		final_pod.depth = 0
		
		for key,value in pairs(rules) do
				local pod, depth = match_rule(option, value)
				if depth ~= 0 then
						if depth > final_pod.depth then
								final_pod.name = pod
								final_pod.depth = depth
						else
								ngx.log(ngx.INFO, "multiple rule match!!")
						end
				end				
		end
		
		ngx.log(ngx.INFO, "final_pod: ", cjson.encode(final_pod))

		local final_upstreams = default_upstreams
		val, err = pods_ngx_cache:get(final_pod.name)
		if val then
				final_upstreams =  string_util.split(val, ";")
		end
		return final_upstreams
end

return rule_engine
```



# setctx.lua

```lua
local string_util = require "string_util"

local ip = ngx.var.remote_addr
local uri = ngx.var.uri
local account = ngx.var.remote_user
ngx.ctx.account = account

--[[
local account = nil
local org_authorization = ngx.req.get_headers()["Authorization"]
if org_authorization then
		local authorization = string.gsub(org_authorization, "Basic ", "")
		authorization = ngx.decode_base64(authorization)
		
		local temp = string_util.split(authorization, ":")
		account = temp[1]
		ngx.ctx.account = account
end
]]

ngx.log(ngx.INFO, "ip :", ip)
ngx.log(ngx.INFO, "uri :", uri)
--ngx.log(ngx.INFO, "authorization :", org_authorization)
ngx.log(ngx.INFO, "account :", account)

local option = {}
option.ip = ip
option.uri = uri
option.account = account

local temps_ngx_cache = ngx.shared.temps
local org_md5_str = ip.."_"..uri
if account then
		org_md5_str = org_md5_str.."_"..account
end
ngx.log(ngx.INFO, "org_md5_str: ", org_md5_str)
local option_md5 = ngx.md5(org_md5_str)
ngx.log(ngx.INFO, "md5_str: ", option_md5)
local upstream_cache, err = temps_ngx_cache:get(option_md5)
local upstreams = ""
if upstream_cache then
		ngx.log(ngx.INFO, "load from cache.......")
		upstreams = string_util.split(upstream_cache, ";")
else
		ngx.log(ngx.INFO, "new caculate.......")
		
		local rules_ngx_cache = ngx.shared.rules
		local rule_items_ngx_cache = ngx.shared.rule_items
		local pods_ngx_cache = ngx.shared.pods
		
		local rule_engine = require "rule_engine"
		upstreams = rule_engine.get_upstreams(option, rules_ngx_cache, rule_items_ngx_cache, pods_ngx_cache)
		-- upstreams = {"192.1.1.1","192.168.27.216","172.16.151.62"}
		
		local ok, err = temps_ngx_cache:set(option_md5, table.concat(upstreams, ";"), 7200)
		if not ok then
				ngx.log(ngx.ERR, "failed to set temp cache for upstreams: ", err)
		end

end

local cjson = require "cjson"
ngx.log(ngx.INFO, "upstreams: ",  cjson.encode(upstreams))

ngx.ctx.upstreams = upstreams
local loop = #upstreams - 1
if loop > 2
then
		loop = 2
end
ngx.ctx.upstream_loop = loop
```

# balance.lua

```lua
local b = require "ngx.balancer"
ngx.log(ngx.INFO, "upstreaming........")

if not ngx.ctx.tries then
		ngx.ctx.tries = 0
end

local upstreams = ngx.ctx.upstreams
ngx.log(ngx.INFO, "total upstream backends: ", #upstreams)

if ngx.ctx.tries < ngx.ctx.upstream_loop then
		local ok, err = b.set_more_tries(1)
		if not ok then
				return error("failed to set more tries: ", err)
		elseif err then
				ngx.log(ngx.WARN, "set more tries: ", err)
		end
end
ngx.ctx.tries = ngx.ctx.tries + 1

math.randomseed(tostring(os.time()):reverse():sub(1, 6))
local index = math.random(1, #upstreams)
ngx.log(ngx.INFO, "choose upstream index: ", index)
local host = upstreams[index]

--local port = ngx.var.server_port
local upstream = require "ngx.upstream"
local org_addr = upstream.get_servers("backend")[1].addr
local string_util = require "string_util"
local addr_arr = string_util.split(org_addr, ":")
local port = addr_arr[2]

ngx.log(ngx.INFO, "tries: ", ngx.ctx.tries, ", host: ", host, ", port: ", port)
local ok, err = b.set_current_peer(host, port)
if not ok then
		ngx.log(ngx.ERR, "failed to set the current peer: ", err)
		return ngx.exit(500)
end

table.remove(upstreams, index)
```

# matcher.lua

```lua
matcher = {}

function matcher.match_equal(value, pattern)
		return value == pattern
end

function matcher.match_contain(value, pattern)
		local i,j = string.find(value, pattern) 
		if i then
				return true
		end
		return false
end

function matcher.match(value, operator, pattern)
		if not value then
				return false
		end
		
		if operator == "equal" then		
				return matcher.match_equal(value, pattern)
				
		elseif operator == "contain" then
				return matcher.match_contain(value, pattern)
				
		end
end

return matcher
```

