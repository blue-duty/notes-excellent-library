# 1. option模式
option模式在我们对于项目的一些配置项进行设置的时候，可以使用option模式，这样可以使得我们的代码更加的简洁，也更加的易读。
并不是每一个配置项都是我们所需要配置的，有些配置项可以同时初始化的配置来简化我们在使用时的配置，在这种情况下，我们可以采用多参数的方式来进行所有项的配置，但在多参数的情况下，我们就需要对于不需要配置的项进行设置为nil，
设置为nil后我们则需要多花好几步去一次一次的判空操作，这样可以解决问题，但不够优雅，也十分影响代码的可读性。
- 例如以下代码：
```go
type Server struct {
	Addr string
	Port int
}
func NewServer(addr string, port int) *Server {
	if port == 0 {   // 这里我们需要判断port是否为0，如果为0则需要设置为默认值
		port = 8080
	}
	return &Server{
		Addr: addr,
		Port: port,
	}
}
```
这里我们可以使用option模式来进行优化，我们可以将port的默认值设置为option模式，这样我们就可以在初始化的时候进行配置，如果不配置则使用默认值，如果配置则使用配置的值。
```go
type Server struct {
    Addr string
    Port int
}
type Option func(*Server)
func NewServer(addr string, opts ...Option) *Server {
    s := &Server{
        Addr: addr,
        Port: 8080,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}
func WithPort(port int) Option {
    return func(s *Server) {
        s.Port = port
    }
}
```
这样我们就可以在初始化的时候进行配置，如果不配置则使用默认值，如果配置则使用配置的值。
```go
s := NewServer(" localhost", WithPort(8080))
```
- 我们看一下gorm的option模式的使用：
```go
// Apply update config to new config
func (c *Config) Apply(config *Config) error {
	if config != c {
		*config = *c
	}
	return nil
}

// AfterInitialize initialize plugins after db connected
func (c *Config) AfterInitialize(db *DB) error {
	if db != nil {
		for _, plugin := range c.Plugins {
			if err := plugin.Initialize(db); err != nil {
				return err
			}
		}
	}
	return nil
}

// Option gorm option interface
type Option interface {
	Apply(*Config) error
	AfterInitialize(*DB) error
}

func Open(dialector Dialector, opts ...Option) (db *DB, err error) {
	config := &Config{}

	sort.Slice(opts, func(i, j int) bool {
		_, isConfig := opts[i].(*Config)
		_, isConfig2 := opts[j].(*Config)
		return isConfig && !isConfig2
	})

	for _, opt := range opts {
		if opt != nil {
			if applyErr := opt.Apply(config); applyErr != nil {
				return nil, applyErr
			}
			defer func(opt Option) {
				if errr := opt.AfterInitialize(db); errr != nil {
					err = errr
				}
			}(opt)
		}
	}

	if d, ok := dialector.(interface{ Apply(*Config) error }); ok {
		if err = d.Apply(config); err != nil {
			return
		}
	}
}
```
其中有两个方法：Apply和AfterInitialize，这两个方法都是在初始化的时候进行调用的，Apply方法是在初始化的时候进行调用的，AfterInitialize方法是在初始化完成后进行调用的。
- Apply方法是在初始化的时候进行调用的，这个方法是用来对配置进行初始化的，我们可以在这个方法中对配置进行初始化，例如设置默认值等。
- AfterInitialize方法是在初始化完成后进行调用的，这个方法是用来对初始化完成后的对象进行操作的，例如初始化插件等。
- 在我看来，Apply方法是用于通过其中的配置来初始化整个项目的配置，而AfterInitialize方法是用于通过各种配置项初始化一个数据库连接对象，然后通过这个对象来进行数据库操作。
也就是说，可以将这两个看作两个单独的option模式，一个是用于初始化项目的配置，一个是用于初始化一个数据库连接对象。


