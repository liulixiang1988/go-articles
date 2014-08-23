#Go中Json对时间的处理

##Go的自带实现

JSON中经常要处理一些时间，Go中时怎样实现的呢？通过查看Go的time.go文件，我们可以找到下面的代码：

	// MarshalJSON implements the json.Marshaler interface.
	// The time is a quoted string in RFC 3339 format, with sub-second precision added if present.
	func (t Time) MarshalJSON() ([]byte, error) {
		if y := t.Year(); y < 0 || y >= 10000 {
			// RFC 3339 is clear that years are 4 digits exactly.
			// See golang.org/issue/4556#c15 for more discussion.
			return nil, errors.New("Time.MarshalJSON: year outside of range [0,9999]")
		}
		return []byte(t.Format(`"` + RFC3339Nano + `"`)), nil
	}

	// UnmarshalJSON implements the json.Unmarshaler interface.
	// The time is expected to be a quoted string in RFC 3339 format.
	func (t *Time) UnmarshalJSON(data []byte) (err error) {
		// Fractional seconds are handled implicitly by Parse.
		*t, err = Parse(`"`+RFC3339+`"`, string(data))
		return
	}

`MarshalJSON`方法是把Time转化成JSON时调用的，使用的是`RFC3339Nano`这种格式（后面会看到这个格式的讲解）。

`UnmarshalJSON`方法是将JSON转化为Time时使用的，使用的是`RFC3339`这种格式。

我们再跳转到RFC3339Nano的定义，会打开format.go这个文件，会看到如下定义：

	const (
		ANSIC       = "Mon Jan _2 15:04:05 2006"
		UnixDate    = "Mon Jan _2 15:04:05 MST 2006"
		RubyDate    = "Mon Jan 02 15:04:05 -0700 2006"
		RFC822      = "02 Jan 06 15:04 MST"
		RFC822Z     = "02 Jan 06 15:04 -0700" // RFC822 with numeric zone
		RFC850      = "Monday, 02-Jan-06 15:04:05 MST"
		RFC1123     = "Mon, 02 Jan 2006 15:04:05 MST"
		RFC1123Z    = "Mon, 02 Jan 2006 15:04:05 -0700" // RFC1123 with numeric zone
		RFC3339     = "2006-01-02T15:04:05Z07:00"
		RFC3339Nano = "2006-01-02T15:04:05.999999999Z07:00"
		Kitchen     = "3:04PM"
		// Handy time stamps.
		Stamp      = "Jan _2 15:04:05"
		StampMilli = "Jan _2 15:04:05.000"
		StampMicro = "Jan _2 15:04:05.000000"
		StampNano  = "Jan _2 15:04:05.000000000"
	)

至此，我们知道了Go对JSON发送过来的数据格式的定义。下面我们来介绍一些如何让Go的JSON解析支持自定义的时间格式。

##自定义JSON的UnmarshalJSON和MarshalJSON

That's great, thanks for the help Roger. 

Here's how I'm Parsing the string : 

	package main 
	import ( 
	    "json" 
	    "time" 
	    "os" 
	    "fmt" 
	    "strings" 
	) 

	const Fmt = "2006-01-02T15:04:05" 
	type jTime time.Time 
	func (jt *jTime) UnmarshalJSON(data []byte) os.Error { 
	    var s string 
	    if err := json.Unmarshal(data, &s); err != nil { 
	        return err 
	    } 
	    t, err := time.Parse(Fmt, s) 
	    if err != nil { 
	        // find last "." 
	        i := strings.LastIndex(s, ".") 
	        if i == -1 { 
	            return err 
	        } 

	        // try to parse the slice of the string 
	        t, err = time.Parse(Fmt, s[0:i]) 
	        if err != nil { 
	            return err 
	        } 
	    } 
	    *jt = (jTime)(*t) 
	    return nil 
	} 

	func (jt jTime) MarshalJSON() ([]byte, os.Error) { 
	    return json.Marshal((*time.Time)(&jt).Format(Fmt)) 
	} 

	type Foo struct { 
	    T *jTime 
	    I int 
	} 

	type Test struct{ 
	    Time *jTime 
	} 

	func main() { 
	    cjson := []byte("{\"t\":\"2010-12-15T03:56:26.650Z\", \"i\":123}") 

	    var foo Foo; 
	    err := json.Unmarshal(cjson, &foo) 

	    if err != nil { 
	        fmt.Println(err.String()) 
	    } 

	    fmt.Println((*time.Time)(foo.T).Format("1 January 2006 @ 3:04 
	pm") ) 

	} 


This leaves me with one issue, having to type the property to call it 
properties, like Format() for example 

	(*time.Time)(foo.T).Format 

	I could add a method 

	func(jt *jTime) GetTime() *time.Time{ 
	   return (*time.Time)(jt.T); 
	} 

But wondered if there was another way to have a property of type 
time.Time that contains the value of the jt.T? 

Many thanks for all your help. 

##参考

- https://groups.google.com/d/msg/golang-dev/I1dGXiwhJaw/rcGVVZDVFcAJ
- http://www.ietf.org/rfc/rfc3339.txt
- http://stackoverflow.com/questions/455580/json-datetime-between-python-and-javascript