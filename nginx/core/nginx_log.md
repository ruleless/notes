# Nginx日志

## 日志接口
Nginx中输出日志的接口为：

  1. `ngx_log_error`  各模块中通过该接口输出日志
  2. `ngx_log_debugX` 调试日志输出接口，在开启了 *NGX_DEBUG* 时才有效，该接口实际通过调用 `ngx_log_debug` 打印调试日志

## 日志等级
Nginx中定义的日志等级有：
``` c
#define NGX_LOG_STDERR 0
#define NGX_LOG_EMERG 1
#define NGX_LOG_ALERT 2
#define NGX_LOG_CRIT 3
#define NGX_LOG_ERR 4
#define NGX_LOG_WARN 5
#define NGX_LOG_NOTICE 6
#define NGX_LOG_INFO 7
#define NGX_LOG_DEBUG 8
```

## 日志接口实现
`ngx_log_error` 是一个宏：
``` c
#define ngx_log_error(level, log, ...)                                        \
    if ((log)->log_level >= level) ngx_log_error_core(level, log, __VA_ARGS__)
```

`ngx_log_error_core` 是实际的日志接口实现函数，其声明为：
``` c
void
ngx_log_error_core(ngx_uint_t level, ngx_log_t *log, ngx_err_t err,
    const char *fmt, va_list args)
```
对其参数的说明如下：

  1. `ngx_uint_t level`  日志等级
  2. `ngx_log_t *log` 日志结构体，该结构体用于定制各模块的日志输出行为
  3. `ngx_err_t err` 错误码，当系统调用发生错误时将 `errno` 通过该参数传递进来
  4. ` const char *fmt` 格式化字符串，同 `printf` 的格式化字符串参数，除了 `printf` 所支持的字符串格式化标记，Nginx还定义了自己独特的格式化标记
  5. `va_list args` 配合 `fmt` 参数使用的可变参数列表

### ngx_log_t日志结构体
结构体定义为：
``` c
struct ngx_log_s {
    ngx_uint_t log_level;
    ngx_open_file_t *file;

    ngx_atomic_uint_t connection;

    time_t disk_full_time;

    ngx_log_handler_pt handler;
    void *data;

    ngx_log_writer_pt writer;
    void  *wdata;

    /*
     * we declare "action" as "char *" because the actions are usually
     * the static strings and in the "u_char *" case we have to override
     * their types all the time
     */

    char *action;

    ngx_log_t *next;
};
```

对其中关键字段的说明如下：

  * `ngx_uint_t log_level` 日志过滤等级，低于该等级的日志不进行输出
  * `ngx_open_file_t *file` 打开的日志文件
  * `ngx_log_handler_pt handler` Nginx允许在每条日志的末尾拼接模块自己的日志信息，业务模块通过实现该回调函数来构造自己的日志信息。如：http模块通过 `ngx_http_log_error` 来将请求的客户端IP、接入的服务端地址、请求行等信息填充到日志的末尾
  * `ngx_log_writer_pt writer` 一般情况下Nginx将日志输出到文件，但如果有特殊要求，也可通过 `writer` 来将日志信息输出到其他目标。如：Nginx中定义了 `ngx_syslog_writer` 函数来将日志输出到 `syslog`
  * `char *action` 该字段需配合 `handler` 字段一起使用，`action` 表示一个业务阶段。一般而言，业务模块会将将该字段拼接到 *while* 短语之后，表示该日志在什么时机下产生
  * `ngx_log_t *next` 指向下一个日志对象，这在日志初始化的时候会设置好。日志一般通过 `nginx.conf` 配置文件中的指令来初始化

### fmt格式化字符串
fmt格式化字符串除了支持 `printf` 的格式化标记外，还有自己独特的字符串格式化标记：

  * `%P` 用于输出 `ngx_pid_t` 类型的变量
  * `%M` 用于输出 `ngx_msec_t` 类型的变量
  * `%V` 用于输出 `ngx_str_t` 类型的变量
  * `%v` 用于输出 `ngx_http_variable_value_t` 类型的变量
