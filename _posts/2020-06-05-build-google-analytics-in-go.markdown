---
layout: post
title: "Build Google Analytics in Go"
date: 2020-06-05 00:07:00 +0000
description: When you think of building Google Analytics what comes to mind? Visualising the enormity of the platform with all the bells and whistles won't help, you'll fail.
tags: [Go]
---

When you think of building Google Analytics what comes to mind? Visualising the enormity of the platform with all the bells and whistles won't help, you'll fail. But hold on, move closer, discard all the adornments and look at the core, what do you see?

Google Analytics at its core is a very simple application, it extracts useful data from the HTTP Request that a user generates when visiting a webpage, to support the additional features it further sends some more data from the JavaScript code through asynchronous HTTP calls.


In this blog post, we'll be building a basic version of Google Analytics in Go, which I'll call `go-gal-analytics` (pronounce however you want). 


The goal of this blog post is to show how simple it is to build something cool in Go. To follow this tutorial you must know at least the basics of Go and willpower to reach the end.

Before we start, have a look at the `go-gal-analytics` [dashboard](https://go-gal.herokuapp.com/dashboard/).  

We'll be tracking **page views** for the following parameters-
* Country
* City
* Device
* Platform
* OS
* Browser
* Language
* Referral

#### How things work

A website consists of many resources like HTML, CSS, JS, Images, etc. and all of these are essentially files which are stored on a server. When a user types in the website address and hits enter, the browser creates an HTTP Request for each of the resources and sends it to the server.
The server then evaluates each of these requests and sends the requested resources back, which we call an HTTP Response. 

The HTTP Request and Response both contains useful information known as [HTTP Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers).

We'll use HTTP Headers of the Request to track the below parameters-
* Country and City - Retrieve user's IP address from `X-Forwarded-For` or `X-Real-Ip` headers, then it can be used to lookup Country/City from Maxmind Geo Database
* Device, Platform, OS, Browser - Parse the `User-Agent` HTTP Header
* Language - Parse the `Accept-Language` HTTP Header
* Referral - Parse the `Referer` HTTP Header

We'll then place a dummy transparent image pixel (1x1 size) on the website that we want to track analytics on. This way the user cannot see the image but the browser will interpret it as a resource and will send an HTTP Request to our server with all the amazing HTTP Headers, we extract the Headers, track the data and send back the dummy transparent image pixel as HTTP Response.

This is our image pixel -
```html
<img src="http://localhost/knock-knock" border="0" width="1" height="1" />
```

Let's start with the project structure-
```bash
$ go-gal-analytics
├── gogal
│   ├── server.go
│   ├── assets
│   │   └── GeoLite2-City.mmdb
│   ├── handler
│   │   └── event.go
│   ├── model
│   │   └── event.go
│   ├── repository
│   │   └── event.go
│   ├── route
│   │   └── route.go
│   ├── service
│   │   └── event.go
│   ├── utils
│   │   └── counter
│   │       └── counter.go
│   └── web
│       ├── css
│       │   └── style.css
│       ├── index.html
│       └── js
│           └── script.js
├── go.mod
├── go.sum
└── main.go 
```

#### Note: 
In my previous tutorial few people faced difficulty following the blog post because they frequently had to look up the package name and imports to be used for each of the `.go` files.

For sake of simplicity I use the **immediate parent folder** name of the `.go` file as the package name consistently.
Example-
* `event.go` file is inside the `service` folder so `package service` is used. 
* `server.go` file is inside the `gogal` folder so `package gogal` is used. 
* `counter.go` file is inside the `counter` folder so `package counter` is used.
* `main.go` is at the root and is the entrypoint of the application so `package main` is used.

As for `import`, use VSCode or some other editor and install the Go plugin, your imports will be added automatically.

## 1: Initialize the project

Create a `go.mod` file-

```bash
$ go mod init
```

## 2: Counter to count Page Views

`counter` is of type `map[string]uint64`, because our page view count for specific parameter will be saved as-  

Country:
```json
{
    "USA": 83723,
    "UK": 2323
}
```
OS:
```json
{
    "Linux": 4324,
    "Windows": 958
}
```
and same for other parameters.  

Create `counter.go` inside the `counter` folder.
```go
package counter

import (
	"sync"
)

// Counter is go routine safe counter used to count events.
// We add Read/Write Mutex to prevent race conditions.
type Counter struct {
	sync.RWMutex
	counter map[string]uint64
}

// NewCounter creates and returns a new Counter
func NewCounter() *Counter {
	return &Counter{
		counter: make(map[string]uint64),
	}
}

// Incr increments counter for specified key
func (c *Counter) Incr(k string) {
	c.Lock()
	c.counter[k]++
	c.Unlock()
}

// Val returns current value for specified key
func (c *Counter) Val(k string) uint64 {
	var v uint64
	c.RLock()
	v = c.counter[k]
	c.RUnlock()
	return v
}

// Items returns all the counter items
func (c *Counter) Items() map[string]uint64 {
	c.RLock()
	items := make(map[string]uint64, len(c.counter))
	for k, v := range c.counter {
		items[k] = v
	}
	c.RUnlock()
	return items
}
```

## 3: Event model

An `Event` is a single Page View with all the parameters that we'll track.  
We use the following model for `Event`-

```go
package model

type Event struct {
	Location EventLocation `json:"location"`
	Device   EventDevice   `json:"device"`
	Referral string        `json:"referral"`
}

type EventLocation struct {
	Country string `json:"country"`
	City    string `json:"city"`
}

type EventDevice struct {
	Type     string `json:"type"`
	Platform string `json:"platform"`
	OS       string `json:"os"`
	Browser  string `json:"browser"`
	Language string `json:"language"`
}

func (e *Event) Valid() bool {
	if e.Location.City != "" ||
		e.Location.Country != "" ||
		e.Device.Type != "" ||
		e.Device.Platform != "" ||
		e.Device.OS != "" ||
		e.Device.Browser != "" {
		return true
	}
	return false
}

``` 


## 4: Event repository 

`EventRepository` is the storage repository we'll use, we are not using a persistent datastore for simplicity sake.

Create `event.go` inside the `repository` folder.

```go
package repository

import (
	"github.com/erybz/go-gal-analytics/gogal/model"
	"github.com/erybz/go-gal-analytics/gogal/utils/counter"
)

// Stats are constants for which event stats can be retrieved
type Stats string

const (
	// StatsLocationCountry is stats for Country
	StatsLocationCountry Stats = "country"
	// StatsLocationCity is stats for City
	StatsLocationCity = "city"
	// StatsDeviceType is stats for Device Type
	StatsDeviceType = "deviceType"
	// StatsDevicePlatform is stats for Device Platform
	StatsDevicePlatform = "devicePlatform"
	// StatsDeviceOS is stats for OS
	StatsDeviceOS = "os"
	// StatsDeviceBrowser is stats for Browser
	StatsDeviceBrowser = "browser"
	// StatsDeviceLanguage is stats for Language
	StatsDeviceLanguage = "language"
	// StatsReferral is stats for Referral
	StatsReferral = "referral"
)

// EventRepository is storage repository for Events.
// We are not using a persistent datastore like SQL.
// Once the application is exited all stats will be gone.
type EventRepository struct {
	locationCountry *counter.Counter
	locationCity    *counter.Counter
	deviceType      *counter.Counter
	devicePlatform  *counter.Counter
	deviceOS        *counter.Counter
	deviceBrowser   *counter.Counter
	deviceLanguage  *counter.Counter
	referral        *counter.Counter
}

// NewEventRepository creates and returns new EventRepository
func NewEventRepository() *EventRepository {
	return &EventRepository{
		locationCountry: counter.NewCounter(),
		locationCity:    counter.NewCounter(),
		deviceType:      counter.NewCounter(),
		devicePlatform:  counter.NewCounter(),
		deviceOS:        counter.NewCounter(),
		deviceBrowser:   counter.NewCounter(),
		deviceLanguage:  counter.NewCounter(),
		referral:        counter.NewCounter(),
	}
}

// AddEvent adds an event to the repository
func (tr *EventRepository) AddEvent(ev *model.Event) {
	tr.locationCountry.Incr(ev.Location.Country)
	tr.locationCity.Incr(ev.Location.City)
	tr.deviceType.Incr(ev.Device.Type)
	tr.devicePlatform.Incr(ev.Device.Platform)
	tr.deviceOS.Incr(ev.Device.OS)
	tr.deviceBrowser.Incr(ev.Device.Browser)
	tr.deviceLanguage.Incr(ev.Device.Language)
	tr.referral.Incr(ev.Referral)
}

// Events returns stats for the specified event query
func (tr *EventRepository) Events(d Stats) map[string]uint64 {
	m := make(map[string]uint64)
	switch d {
	case StatsLocationCountry:
		m = tr.locationCountry.Items()
	case StatsLocationCity:
		m = tr.locationCity.Items()
	case StatsDeviceType:
		m = tr.deviceType.Items()
	case StatsDevicePlatform:
		m = tr.devicePlatform.Items()
	case StatsDeviceOS:
		m = tr.deviceOS.Items()
	case StatsDeviceBrowser:
		m = tr.deviceBrowser.Items()
	case StatsDeviceLanguage:
		m = tr.deviceLanguage.Items()
	case StatsReferral:
		m = tr.referral.Items()
	}
	return m
}

```

## 5: Create event service

`EventService` connects to the `EventRepository`.  

It builds `Event` from HTTP Request and stores it inside the `EventRepository`.  
It also queries the `EventRepository` for stats.

Create `event.go` inside the `service` folder.

```go
package service

import (
	"log"
	"net"
	"net/http"
	"net/url"

	"github.com/avct/uasurfer"
	"github.com/erybz/go-gal-analytics/gogal/model"
	"github.com/erybz/go-gal-analytics/gogal/repository"
	"github.com/oschwald/geoip2-golang"
	"github.com/tomasen/realip"
	"golang.org/x/text/language"
)

// EventService is service for event logging and stats
type EventService struct {
	eventRepo   *repository.EventRepository
	geoIPReader *geoip2.Reader
}

// NewEventService returns new EventService
func NewEventService() *EventService {
	return &EventService{
		eventRepo:   repository.NewEventRepository(),
		geoIPReader: initGeoIPReader("gogal/assets/GeoLite2-City.mmdb"),
	}
}

// BuildEvent builds a trackable event from the request
func (ts *EventService) BuildEvent(r *http.Request) (*model.Event, error) {
	clientIP := net.ParseIP(realip.FromRequest(r))
	userAgent := uasurfer.Parse(r.UserAgent())
	referrerURL, _ := url.Parse(r.Referer())
	langTags, _, _ := language.ParseAcceptLanguage(r.Header.Get("Accept-Language"))

	userLanguage := ""
	if langTags != nil && len(langTags) >= 1 {
		userLanguage = langTags[0].String()
	}

	geoData, err := ts.geoIPReader.City(clientIP)
	if err != nil {
		return nil, err
	}

	if userAgent.IsBot() {
		return nil, nil
	}

	event := &model.Event{
		Location: model.EventLocation{
			Country: geoData.Country.Names["en"],
			City:    geoData.City.Names["en"],
		},
		Device: model.EventDevice{
			Type:     userAgent.DeviceType.StringTrimPrefix(),
			Platform: userAgent.OS.Platform.StringTrimPrefix(),
			OS:       userAgent.OS.Name.StringTrimPrefix(),
			Browser:  userAgent.Browser.Name.StringTrimPrefix(),
			Language: userLanguage,
		},
		Referral: referrerURL.Hostname(),
	}
	return event, nil
}

// LogEvent logs the event to repository
func (ts *EventService) LogEvent(event *model.Event) {
	ts.eventRepo.AddEvent(event)
}

// Stats retrieves event statistics from the repository
// Since we are storing the events as-
// {
//     "USA": 83723,
//     "UK": 2323
// }
// we need to convert it as follows for the Stats API-
// [
//   {
//     "country": "USA",
//     "pageViews": 83723
//   },
//   {
//     "country": "UK",
//     "pageViews": 2323
//   }
// ]

func (ts *EventService) Stats(dim repository.Stats) []map[string]interface{} {
	allStats := make([]map[string]interface{}, 0, 1)
	for k, v := range ts.eventRepo.Events(dim) {
		stat := map[string]interface{}{
			string(dim): k,
			"pageViews": v,
		}
		allStats = append(allStats, stat)
	}
	return allStats
}

func initGeoIPReader(path string) *geoip2.Reader {
	db, err := geoip2.Open(path)
	if err != nil {
		log.Fatal(err)
	}
	return db
}
```

## 6. Create the Handler

`EventHandler` uses the `EventService`.  

It handles the HTTP Request for tracking the event and providing stats.

Create `event.go` inside the `handler` folder.


```go
package handler

import (
	"encoding/json"
	"log"
	"net/http"

	"github.com/erybz/go-gal-analytics/gogal/repository"
	"github.com/erybz/go-gal-analytics/gogal/service"
	"github.com/julienschmidt/httprouter"
)

// EventHandler is handler for Events
type EventHandler struct {
	eventService *service.EventService
}

// NewEventHandler creates and returns new EventHandler
func NewEventHandler() *EventHandler {
	return &EventHandler{
		eventService: service.NewEventService(),
	}
}

// Track accepts analytics request and builds event from it
func (h *EventHandler) Track(w http.ResponseWriter, r *http.Request, _ httprouter.Params) {
	if r.Method != http.MethodGet {
		http.Error(w, "Request method is not GET", http.StatusNotFound)
		return
	}
	event, err := h.eventService.BuildEvent(r)
	if err != nil {
		log.Println(err)
	}

	if event != nil && event.Valid() {
		h.eventService.LogEvent(event)
	}

	w.Header().Set("Cache-Control", "no-cache, no-store, must-revalidate")
	w.Header().Set("Content-Type", "image/gif")
	w.Write(createPixel())
}

// Stats retrieves stats for the specified query
func (h *EventHandler) Stats(w http.ResponseWriter, r *http.Request, _ httprouter.Params) {
	if r.Method != http.MethodGet {
		http.Error(w, "Request method is not GET", http.StatusNotFound)
		return
	}

	urlVals := r.URL.Query()
	query := urlVals.Get("q")

	stats := h.eventService.Stats(
		repository.Stats(query),
	)

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(stats)
}

// This is an image pixel, specifically GIF89a
// Check the references at the end for more info. 
func createPixel() []byte {
	return []byte{
		71, 73, 70, 56, 57, 97, 1, 0, 1, 0, 128, 0, 0, 0, 0, 0,
		255, 255, 255, 33, 249, 4, 1, 0, 0, 0, 0, 44, 0, 0, 0, 0,
		1, 0, 1, 0, 0, 2, 1, 68, 0, 59,
	}
}
```

## 7. Create routes
Add `route.go` inside the `route` directory

```go
package route

import (
	"net/http"

	"github.com/erybz/go-gal-analytics/gogal/handler"
	"github.com/julienschmidt/httprouter"
)

// Routes initializes the routes
func Routes() http.Handler {
	rt := httprouter.New()

	eventHandler := handler.NewEventHandler()
	rt.GET("/knock-knock", eventHandler.Track)
	rt.GET("/stats", eventHandler.Stats)

	rt.ServeFiles("/dashboard/*filepath", http.Dir("./gogal/web"))

	return rt
}

```

## 8. Initialize the server

Add `server.go` inside the `gogal` directory

```go
package gogal

import (
	"fmt"
	"log"
	"net/http"
)

// Server struct containing hostname and port
type Server struct {
	Hostname string `json:"hostname"`
	HTTPPort string `json:"httpPort"`
}

// NewServer creates new instance of server
func NewServer(host, port string) *Server {
	return &Server{
		Hostname: host,
		HTTPPort: port,
	}
}

// Run starts the server at specified host and port
func (s *Server) Run(h http.Handler) {
	fmt.Println(s.Message())

	log.Printf("Listening at %s", s.Address())
	log.Fatal(http.ListenAndServe(s.Address(), h))
}

// Address returns formatted hostname and port
func (s *Server) Address() string {
	return fmt.Sprintf("%s:%s", s.Hostname, s.HTTPPort)
}

// Message is the server start message
func (s *Server) Message() string {
	m := `
                                      .__   
   ____   ____             _________  |  |  
  / ___\ /  _ \   ______  / ___\__  \ |  |  
 / /_/  (  <_> ) /_____/ / /_/  / __ \|  |__
 \___  / \____/          \___  (____  |____/
/_____/                 /_____/     \/      
                                     Analytics

`
	return m
}

```

## 9. Add the application entrypoint
Create `main.go` at the root directory

```go
package main

import (
	"flag"

	"github.com/erybz/go-gal-analytics/gogal"
	"github.com/erybz/go-gal-analytics/gogal/route"
)

func main() {
	hostname := flag.String(
		"h", "0.0.0.0", "hostname",
	)
	port := flag.String(
		"p", "8000", "port",
	)
	flag.Parse()

	s := gogal.NewServer(*hostname, *port)
	r := route.Routes()
	s.Run(r)
}
```

## 10. Run it!
```bash
$ go run main.go -p 80

                                      .__   
   ____   ____             _________  |  |  
  / ___\ /  _ \   ______  / ___\__  \ |  |  
 / /_/  (  <_> ) /_____/ / /_/  / __ \|  |__
 \___  / \____/          \___  (____  |____/
/_____/                 /_____/     \/      
                                     Analytics


2020/06/05 00:00:50 Listening at 0.0.0.0:80
```

Place this tracker on a website-
```html
<img src="http://localhost/knock-knock" border="0" width="1" height="1" />
```

Query the Stats API for stats-
```bash
curl -s -X GET "https://go-gal.herokuapp.com/stats?q=country"
```
```json
[
  {
    "country": "United States",
    "pageViews": 5
  },
  {
    "country": "Canada",
    "pageViews": 3
  }
]
```

Following are the APIs currently provided by `go-gal-analytics`-
* /knock-knock - Track events, use inside `<img>` tag
* /stats - Retrieve Page Views stats for the following parameters-
  * ?q=country
  * ?q=city
  * ?q=deviceType
  * ?q=devicePlatform
  * ?q=os
  * ?q=browser
  * ?q=language
  * ?q=referral

Build a custom UI based on the above APIs, place the files inside the `web` directory which is being served by the Go server itself at the route `/dashboard`.

#### Note: 
The application won't run unless you register on [MaxMind](https://www.maxmind.com) and download the `GeoLite2-City.mmdb` GeoIP database and put it inside the assets folder.

### Further Improvements
* The current implementation only supports tracking a single website, you can create unique tracking URLs and make few changes to support multiple website analytics
* Add a real datastore repository using your own choice of SQL/NoSQL database
* Support tracking other metrics along with Page Views
* Improve the codebase by handling errors gracefully

Find the complete source code in the Github Repository - [go-gal-analytics](https://github.com/erybz/go-gal-analytics)

### Bonus
You can attach the image pixel tracker to your emails and find out the location, browser, etc. of the person who opened your email. Tweak the code and you'll know the exact time when the email was opened.

### References
* [Build a Protocol Buffer Powered Tracking Pixel in Go](https://product.reverb.com/build-a-protocol-buffer-powered-tracking-pixel-in-go-76f2ca5c26e2)
* [All About GIF89a](http://www6.uniovi.es/gifanim/gifabout.htm)
* [sync.Mutex](https://tour.golang.org/concurrency/9)
* [Correct way of getting Client's IP Addresses from http.Request](https://stackoverflow.com/questions/27234861/correct-way-of-getting-clients-ip-addresses-from-http-request)
* [Remote IP Address with Go](https://husobee.github.io/golang/ip-address/2015/12/17/remote-ip-go.html)
* [Get the IP address from an incoming HTTP request](https://golangbyexample.com/golang-ip-address-http-request)
* [How can I detect real location of the user through their IP address?](https://security.stackexchange.com/questions/108885/how-can-i-detect-real-location-of-the-user-through-their-ip-address)
* [How can I determine the location of a visitor to my website?](https://gis.stackexchange.com/questions/88/how-can-i-determine-the-location-of-a-visitor-to-my-website)
* [What are Content-Language and Accept-Language?](https://stackoverflow.com/questions/6157485/what-are-content-language-and-accept-language)
* [Language and Locale Matching in Go](https://blog.golang.org/matchlang)
* [How to get actual localization/lang in golang?](https://stackoverflow.com/questions/39267486/how-to-get-actual-localization-lang-in-golang)