---
layout: post
title: "Diving into Go by building a CLI application"
date: 2020-05-27 00:00:00 +0000
description: You have wrapped your head around the Go syntax and practised them one by one, however you won't feel comfortable writing applications in Go unless you build one.
tags: [Go]
---

[Russian Version](https://howtorecover.me/pogruzhenie-v-go-putem-sozdaniya-prilozheniya-cli)

You have wrapped your head around the Go syntax and practised them one by one, however you won't feel comfortable writing applications in Go unless you build one.

In this blog post we'll build a CLI application in Go, which we'll call **go-grab-xkcd**.
This application fetches comics from [XKCD](https://xkcd.com/) and provides you with various options through command-line arguments.

We'll use no external dependencies and will build the entire app using only the Go standard library.

The application idea looks silly but the aim is to get comfortable writing production (sort of) code in Go and not to get acquired by Google.
###### There is also a Bash Bonus at the end.  
_Note: This post assumes that the reader is familiar with Go syntax and terminologies and is somewhere between a beginner and an intermediate._

Let's first run the application and see it in action-

```bash
$ go-grab-xkcd --help

Usage of go-grab-xkcd:
  -n int
        Comic number to fetch (default latest)
  -o string
        Print output in format: text/json (default "text")
  -s    Save image to current directory
  -t int
        Client timeout in seconds (default 30)
```

```bash
$ go-grab-xkcd -n 323

Title: Ballmer Peak
Comic No: 323
Date: 1-10-2007
Description: Apple uses automated schnapps IVs.
Image: https://imgs.xkcd.com/comics/ballmer_peak.png
```

```bash
$ go-grab-xkcd -n 323 -o json
{
  "title": "Ballmer Peak",
  "number": 323,
  "date": "1-10-2007",
  "description": "Apple uses automated schnapps IVs.",
  "image": "https://imgs.xkcd.com/comics/ballmer_peak.png"
}
```

You can try rest of the options by downloading and running the application for your computer.

After the end of this tutorial you'll be comfortable with the following topics-

1. Accepting command line arguments
2. Interconversion between JSON and Go Structs
3. Making API calls
4. Creating files (Downloading and saving from Internet)
5. String Manipulation

Below is the project structure-

```bash
$ tree go-grab-xkcd
go-grab-xkcd
├── client
│   └── xkcd.go
└── model
    └── comic.go
├── main.go
└── go.mod
```

- `go.mod` - _Go Modules_ file used in Go for package management
- `main.go` - Main entrypoint of the application
- `comic.go` - Go representation of the data as a `struct` and operations on it
- `xkcd.go` - xkcd client for making HTTP calls to the API, parsing response and saving to disk

## 1: Initialize the project

Create a `go.mod` file-

```bash
$ go mod init
```

This will help in package management (think package.json in JS).

## 2: xkcd API

xkcd is amazing, you don't require any signups or access keys to use their API.
Open the xkcd [API "documentation"](https://xkcd.com/json.html) and you'll find that there are 2 endpoints-

1. `http://xkcd.com/info.0.json` - GET latest comic
2. `http://xkcd.com/614/info.0.json` - GET specific comic by comic number

Following is the JSON response from these endpoints-

```json
{
  "num": 2311,
  "month": "5",
  "day": "25",
  "year": "2020",
  "title": "Confidence Interval",
  "alt": "The worst part is that's the millisigma interval.",
  "img": "https://imgs.xkcd.com/comics/confidence_interval.png",
  "safe_title": "Confidence Interval",
  "link": "",
  "news": "",
  "transcript": ""
}
```

Relevant [xkcd](https://xkcd.com/1481/)

## 2: Create model for the Comic

Based on the above JSON response, we create a `struct` called `ComicResponse` in `comic.go` inside the `model` package

```go
type ComicResponse struct {
	Month      string `json:"month"`
	Num        int    `json:"num"`
	Link       string `json:"link"`
	Year       string `json:"year"`
	News       string `json:"news"`
	SafeTitle  string `json:"safe_title"`
	Transcript string `json:"transcript"`
	Alt        string `json:"alt"`
	Img        string `json:"img"`
	Title      string `json:"title"`
	Day        string `json:"day"`
}
```

You can use the [JSON-to-Go](https://mholt.github.io/json-to-go/) tool to automatically generate the struct from JSON.

Also create another struct which will be used to output data from our application.

```go
type Comic struct {
	Title       string `json:"title"`
	Number      int    `json:"number"`
	Date        string `json:"date"`
	Description string `json:"description"`
	Image       string `json:"image"`
}
```

Add the below two methods to `ComicResponse` struct-

```go
// FormattedDate formats individual date elements into a single string
func (cr ComicResponse) FormattedDate() string {
	return fmt.Sprintf("%s-%s-%s", cr.Day, cr.Month, cr.Year)
}
```

```go
// Comic converts ComicResponse that we receive from the API to our application's output format, Comic
func (cr ComicResponse) Comic() Comic {
	return Comic{
		Title:       cr.Title,
		Number:      cr.Num,
		Date:        cr.FormattedDate(),
		Description: cr.Alt,
		Image:       cr.Img,
	}
}
```

Then add the following two methods to the `Comic` struct-

```go
// PrettyString creates a pretty string of the Comic that we'll use as output
func (c Comic) PrettyString() string {
	p := fmt.Sprintf(
		"Title: %s\nComic No: %d\nDate: %s\nDescription: %s\nImage: %s\n",
		c.Title, c.Number, c.Date, c.Description, c.Image)
	return p
}
```

```go
// JSON converts the Comic struct to JSON, we'll use the JSON string as output
func (c Comic) JSON() string {
	cJSON, err := json.Marshal(c)
	if err != nil {
		return ""
	}
	return string(cJSON)
}
```

## 3: Setup xkcd client for making request, parsing response and saving to disk

Create `xkcd.go` file inside the `client` package.

First define a custom type called `ComicNumber` as an `int`

```go
type ComicNumber int
```

Define constants-

```go
const (
	// BaseURL of xkcd
	BaseURL string = "https://xkcd.com"
	// DefaultClientTimeout is time to wait before cancelling the request
	DefaultClientTimeout time.Duration = 30 * time.Second
	// LatestComic is the latest comic number according to the xkcd API
	LatestComic ComicNumber = 0
)
```

Create a struct `XKCDClient`, it will be used to make requests to the API.

```go
// XKCDClient is the client for XKCD
type XKCDClient struct {
	client  *http.Client
	baseURL string
}

// NewXKCDClient creates a new XKCDClient
func NewXKCDClient() *XKCDClient {
	return &XKCDClient{
		client: &http.Client{
			Timeout: DefaultClientTimeout,
		},
		baseURL: BaseURL,
	}
}
```

Add the following 4 methods to `XKCDClient`-

1. `SetTimeout()`

   ```go
   // SetTimeout overrides the default ClientTimeout
   func (hc *XKCDClient) SetTimeout(d time.Duration) {
       hc.client.Timeout = d
   }
   ```

2. `Fetch()`

   ```go
   // Fetch retrieves the comic as per provided comic number
   func (hc *XKCDClient) Fetch(n ComicNumber, save bool) (model.Comic, error) {
       resp, err := hc.client.Get(hc.buildURL(n))
       if err != nil {
           return model.Comic{}, err
       }
       defer resp.Body.Close()

       var comicResp model.ComicResponse
       if err := json.NewDecoder(resp.Body).Decode(&comicResp); err != nil {
           return model.Comic{}, err
       }

       if save {
           if err := hc.SaveToDisk(comicResp.Img, "."); err != nil {
               fmt.Println("Failed to save image!")
           }
       }
       return comicResp.Comic(), nil
   }
   ```

3. `SaveToDisk()`

   ```go
   // SaveToDisk downloads and saves the comic locally
   func (hc *XKCDClient) SaveToDisk(url, savePath string) error {
       resp, err := http.Get(url)
       if err != nil {
           return err
       }
       defer resp.Body.Close()

       absSavePath, _ := filepath.Abs(savePath)
       filePath := fmt.Sprintf("%s/%s", absSavePath, path.Base(url))

       file, err := os.Create(filePath)
       if err != nil {
           return err
       }
       defer file.Close()

       _, err = io.Copy(file, resp.Body)
       if err != nil {
           return err
       }
       return nil
   }
   ```

4. `buildURL()`
   ```go
   func (hc *XKCDClient) buildURL(n ComicNumber) string {
       var finalURL string
       if n == LatestComic {
           finalURL = fmt.Sprintf("%s/info.0.json", hc.baseURL)
       } else {
           finalURL = fmt.Sprintf("%s/%d/info.0.json", hc.baseURL, n)
       }
       return finalURL
   }
   ```

## 4: Connect everything

Inside the `main()` function we connect all the wires-

- Read command arguments
- Instantiate the `XKCDClient`
- Fetch from API using the `XKCDClient`
- Output

##### Read command arguments-

```go
comicNo := flag.Int(
    "n", int(client.LatestComic), "Comic number to fetch (default latest)",
)
clientTimeout := flag.Int64(
    "t", int64(client.DefaultClientTimeout.Seconds()), "Client timeout in seconds",
)
saveImage := flag.Bool(
    "s", false, "Save image to current directory",
)
outputType := flag.String(
    "o", "text", "Print output in format: text/json",
)
flag.Parse()
```

##### Instantiate the `XKCDClient`

```go
xkcdClient := client.NewXKCDClient()
xkcdClient.SetTimeout(time.Duration(*clientTimeout) * time.Second)
```

##### Fetch from API using the `XKCDClient`

```go
comic, err := xkcdClient.Fetch(client.ComicNumber(*comicNo), *saveImage)
if err != nil {
    log.Println(err)
}
```

##### Output

```go
if *outputType == "json" {
    fmt.Println(comic.JSON())
} else {
    fmt.Println(comic.PrettyString())
}
```

Run the program as follows-

```bash
$ go run main.go -n 323 -o json
```

Or build it as an executable binary for your laptop and then run-

```bash
$ go build .
$ ./go-grab-xkcd -n 323 -s -o json
```

Find the complete source code in the Github Repository - [go-grab-xkcd](https://github.com/erybz/go-grab-xkcd)

## Bash Bonus

Download multiple comics serially by using this simple shell magic-

```bash
$ for i in {1..10}; do ./go-grab-xkcd -n $i -s; done;
```

The above shell code simple calls our `go-grab-xkcd` command in a `for` loop, and the `i` value is substituted as comic number since xkcd uses serial integers as comic number/ID.
