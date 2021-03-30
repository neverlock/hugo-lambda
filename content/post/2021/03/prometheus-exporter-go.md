+++
title= "Prometheus Exporter Go"
date= 2021-03-30T17:23:51+07:00
draft=false
og_image = "2021/03/prometheus-exporter-go.jpeg"
categories = ["api","golang","prometheus"]
tags = [
"golang",
"prometheus",
]
+++

## การเขียน prometheus exporter โดยภาษา go
ปรกติแล้วการเขียน exporter ของ prometheus จะมีการเขียน 2 แบบคือ
- แบบที่ตัว function จะทำงานทุกครั้งเมื่อมีการเรียก url ของ exporter 
- แบบที่ตัว function ทำงานเป็น background ตลอดเวลา

แบบแรกจะเหมะสำหรับงานที่ทำงานได้เร็วดึง data ออกมาแสดงผลได้เร็ว ไม่จำเป็นต้องนับค่า data เหล่านั้นสะสมไปเรื่อยๆ เช่นการเก็บ % การใช้งาน cpu ณ. ช่วงเวลาไดๆ

แบบที่ 2 จะเหมาะสำหรับงานที่ต้องการจะดึงข้อมูลเยอะๆแล้วทำงานช้าให้ระบบทำการเก็บข้อมูลไว้รอตลอดเวลาเมื่อมีการ request http มาหา exporter จะสามารถแสดงข้อมูลได้ทันที และงานที่จะต้องเก็บค่า แบบ increse ขึ้นเรื่อยๆระบบนี้จะทำการเก็บข้อมูลดังกล่าวเป็น background ตลอดเวลา

## ตัวอย่างการเขียน exporter แบบแรก
```golang
package main

//Exporter is an important part of the monitoring system implemented based on Prometheus. It is responsible for the collection of data indicators. The official exporter list already contains most common system indicator monitoring, such as node_exporter for machine performance monitoring, Snmp_exporter for network equipment monitoring, etc. For monitoring, these existing exporters can provide complete data indicator collection with little configuration work.
//Sometimes we need to write some indicator monitoring related to business logic. These indicators cannot be obtained through common exporters. For example, we need to provide overall monitoring of DNS resolution. Understanding how to write exporters is very important for business monitoring, and it is also a stage that needs to go through to improve the monitoring system. Next we will introduce how to write exporter.

import (
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
	"log"
	"net/http"
)

// define indicators
// First, define two indicator data, one is Guage type and the other is Counter type. They represent the statistics of CPU temperature and disk failure times, respectively, and are classified using the above definition.
var cpuTemp = prometheus.NewGauge(prometheus.GaugeOpts{
	Name: "cpu_temperature_celsius",
	Help: "Current temperature of the CPU",
})

// Other parameters can also be registered here. For example, in the statistics of the number of disk failures above, we can pass a device name in at the same time, so that we can obtain multiple different indicators when collecting. Each indicator corresponds to the number of disk failures of a device.
var hdFailures = prometheus.NewCounterVec(
	prometheus.CounterOpts{
		Name: "hd_errors_total",
		Help: "Number of hard-disk errors.",
	},
	[]string{"device"},
)

// Registration indicator:
func init() {
	prometheus.MustRegister(cpuTemp)
	prometheus.MustRegister(hdFailures)
}

//Using prometheus.MustRegister is to directly register the data to the Default Registry. Just like the running example above, this Default Registry can pass indicators out without any additional code. After registration, you can use the indicator at the program level. Here we use the API (Set and With().Inc) provided by the previously defined indicator to change the data content of the indicator

func main() {
	cpuTemp.Set(65.3)
	hdFailures.With(prometheus.Labels{"device": "/dev/sda"}).Inc()

	// The Handler function provides a default handler to expose metrics
	// via an HTTP server. "/metrics" is the usual endpoint for that.
	http.Handle("/metrics", promhttp.Handler())
	log.Fatal(http.ListenAndServe(":8080", nil))

}
```

## ตัวอย่างการเขียน exporter แบบที่ 2
```golang
package main

import (
	"crypto/tls"
	"fmt"
	"io/ioutil"
	"net/http"
	"os"
	"strings"

	"github.com/neverlock/openvidu-exporter/session"
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
	"github.com/prometheus/common/log"
	"github.com/spf13/viper"
)

var (
	SESSION_API_ENDPOINT = "/openvidu/api/sessions"

	tr = &http.Transport{
		TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
	}
	client = &http.Client{Transport: tr}
)

type ViduExporter struct {
	Host               string
	User               string
	Pass               string
	ViduSessionAPIDesc *prometheus.Desc //Session or Connection
}

func (v *ViduExporter) ReallyExpensiveAssessmentOfTheSystemState() (objectByHost map[string]int) {
	session, connection, err := getSessionNumber(v.Host, v.User, v.Pass)
	if err != nil {
		return
	}
	objectByHost = map[string]int{
		"session":    session,
		"connection": connection,
	}
	return
}

func getSessionNumber(host string, user string, pass string) (int, int, error) {
	req, err := http.NewRequest("GET", host+SESSION_API_ENDPOINT, nil)
	if err != nil {
		return 0, 0, err
	}

	req.SetBasicAuth(user, pass)

	resp, err := client.Do(req)
	if err != nil {
		return 0, 0, err
	}

	body, err := ioutil.ReadAll(resp.Body)
	resp.Body.Close()
	if err != nil {
		return 0, 0, err
	}
	return session.GetSessionActive(body), session.GetConnectionActive(body), nil
}

func (v *ViduExporter) Describe(ch chan<- *prometheus.Desc) {
	ch <- v.ViduSessionAPIDesc
}

func (v *ViduExporter) Collect(ch chan<- prometheus.Metric) {
	objectByHost := v.ReallyExpensiveAssessmentOfTheSystemState()
	for object, scount := range objectByHost {
		ch <- prometheus.MustNewConstMetric(
			v.ViduSessionAPIDesc,
			prometheus.GaugeValue,
			float64(scount),
			object,
		)
	}
}

func NewExporter(viduHost string, viduUsername string, viduPassword string) *ViduExporter {
	return &ViduExporter{
		Host: viduHost,
		User: viduUsername,
		Pass: viduPassword,
		ViduSessionAPIDesc: prometheus.NewDesc(
			"vidu_session_usage",
			"Number of session use in OpenVidu server",
			[]string{"object"},
			prometheus.Labels{"host": viduHost},
		),
	}

}

func readConfig() {
	viper.SetConfigName("config")
	viper.AddConfigPath(".")
	viper.AutomaticEnv()
	viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
	err := viper.ReadInConfig()
	if err != nil {
		panic(fmt.Errorf("fatal error config file: %s \n", err))
	}
}

func main() {
	readConfig()

	exporterListen := viper.GetString("Vidu.ListenHost") + viper.GetString("Vidu.ListenPort")

	reg := prometheus.NewPedanticRegistry()

	for i := 1; i <= viper.GetInt("Vidu.MaxSub"); i++ {
		sub := fmt.Sprintf("Vidu.Vidu%d", i)
		Vidu := viper.Sub(sub)
		if Vidu == nil {
			panic("Vidu config not found")
		}
		exporter := NewExporter(Vidu.GetString("URL"), Vidu.GetString("User"), Vidu.GetString("Password"))
		reg.MustRegister(exporter)
	}

	gatherers := prometheus.Gatherers{
		prometheus.DefaultGatherer,
		reg,
	}
	h := promhttp.HandlerFor(gatherers,
		promhttp.HandlerOpts{
			ErrorLog:      log.NewErrorLogger(),
			ErrorHandling: promhttp.ContinueOnError,
		})
	http.HandleFunc("/metrics", func(w http.ResponseWriter, r *http.Request) {
		h.ServeHTTP(w, r)
	})
	log.Infoln("Start server at ", exporterListen)
	if err := http.ListenAndServe(exporterListen, nil); err != nil {
		log.Errorf("Error occur when start server %v", err)
		os.Exit(1)
	}

}
```

** อธิบายการเขียนแบบที่ 2 

ตัวอย่างนี้เป็นการเขียน exporter เพื่อดึงค่าจาก openvidu session api 
เริ่มต้นด้วยการออกแบบให้ exporter รับ input ของ url ,user และ password ของ target ที่จะเก็บค่าผ่านทาง yaml โดยใช้ lib viper

```yaml
Vidu:
   ListenHost: 0.0.0.0
   ListenPort: :8080
   MaxSub: 3
   Vidu1:
      URL: https://url1
      User: user1
      Password: pass1
   Vidu2:
      URL: https://url2
      User: user2
      Password: pass2
   Vidu3:
      URL: https://url3
      User: user3
      Password: pass3
```

* จากนั้น สร้าง function NewExporter เพื่อ ini exporter

```golang
func NewExporter(viduHost string, viduUsername string, viduPassword string) *ViduExporter {
	return &ViduExporter{
		Host: viduHost,
		User: viduUsername,
		Pass: viduPassword,
		ViduSessionAPIDesc: prometheus.NewDesc(
			"vidu_session_usage",
			"Number of session use in OpenVidu server",
			[]string{"object"},
			prometheus.Labels{"host": viduHost},
		),
	}

}
```

* ใน function NewExporter จะ return struct ViduExporter ซึ่ง struct นี้จะเก็บข้อมูลต่างๆที่เราต้องการและเราจะฝาก host,user และ password ไว้ใน struct นี้ ด้วยเพื่อนำส่งไปที่ function สำหรับดึงค่าจาก api ในแต่ละ host 
```golang
type ViduExporter struct {
	Host               string
	User               string
	Pass               string
	ViduSessionAPIDesc *prometheus.Desc //Session or Connection
}
```

* จากนั้น Implement Describe interface
```golang
func (v *ViduExporter) Describe(ch chan<- *prometheus.Desc) {
	ch <- v.ViduSessionAPIDesc
}
```

* และ Implement Collect Interface
```golang
func (v *ViduExporter) Collect(ch chan<- prometheus.Metric) {
	objectByHost := v.ReallyExpensiveAssessmentOfTheSystemState()
	for object, scount := range objectByHost {
		ch <- prometheus.MustNewConstMetric(
			v.ViduSessionAPIDesc,
			prometheus.GaugeValue,
			float64(scount),
			object,
		)
	}
}
```

* และ Implement function ที่ทำหน้าที่ collect data ให้เรา
```golang
func (v *ViduExporter) ReallyExpensiveAssessmentOfTheSystemState() (objectByHost map[string]int) {
	session, connection, err := getSessionNumber(v.Host, v.User, v.Pass)
	if err != nil {
		return
	}
	objectByHost = map[string]int{
		"session":    session,
		"connection": connection,
	}
	return
}

func getSessionNumber(host string, user string, pass string) (int, int, error) {
	req, err := http.NewRequest("GET", host+SESSION_API_ENDPOINT, nil)
	if err != nil {
		return 0, 0, err
	}

	req.SetBasicAuth(user, pass)

	resp, err := client.Do(req)
	if err != nil {
		return 0, 0, err
	}

	body, err := ioutil.ReadAll(resp.Body)
	resp.Body.Close()
	if err != nil {
		return 0, 0, err
	}
	return session.GetSessionActive(body), session.GetConnectionActive(body), nil
}
```

* ใน function สำหรับ collect data นั้นจะเรียกใช้ package session ที่ทำหน้าที่ parser json ของ vidu api  โดยมี code ประมาณนี้
```golang
package session

import jsoniter "github.com/json-iterator/go"

type ViduSession struct {
	Numberofelements int `json:"numberOfElements"`
	Content          []struct {
		ID                string `json:"id"`
		Object            string `json:"object"`
		Sessionid         string `json:"sessionId"`
		Createdat         int64  `json:"createdAt"`
		Mediamode         string `json:"mediaMode"`
		Recordingmode     string `json:"recordingMode"`
		Defaultoutputmode string `json:"defaultOutputMode"`
		Customsessionid   string `json:"customSessionId"`
		Connections       struct {
			Numberofelements int `json:"numberOfElements"`
			Content          []struct {
				ID                      string      `json:"id"`
				Object                  string      `json:"object"`
				Status                  string      `json:"status"`
				Connectionid            string      `json:"connectionId"`
				Sessionid               string      `json:"sessionId"`
				Createdat               int64       `json:"createdAt"`
				Activeat                int64       `json:"activeAt"`
				Location                string      `json:"location"`
				Platform                string      `json:"platform"`
				Token                   interface{} `json:"token"`
				Type                    string      `json:"type"`
				Record                  bool        `json:"record"`
				Role                    interface{} `json:"role"`
				Kurentooptions          interface{} `json:"kurentoOptions"`
				Rtspuri                 string      `json:"rtspUri"`
				Adaptativebitrate       bool        `json:"adaptativeBitrate"`
				Onlyplaywithsubscribers bool        `json:"onlyPlayWithSubscribers"`
				Networkcache            int         `json:"networkCache"`
				Serverdata              string      `json:"serverData"`
				Clientdata              interface{} `json:"clientData"`
				Publishers              []struct {
					Createdat    int64  `json:"createdAt"`
					Streamid     string `json:"streamId"`
					Rtspuri      string `json:"rtspUri"`
					Mediaoptions struct {
						Hasaudio        bool        `json:"hasAudio"`
						Audioactive     bool        `json:"audioActive"`
						Hasvideo        bool        `json:"hasVideo"`
						Videoactive     bool        `json:"videoActive"`
						Typeofvideo     string      `json:"typeOfVideo"`
						Framerate       interface{} `json:"frameRate"`
						Videodimensions interface{} `json:"videoDimensions"`
						Filter          struct {
						} `json:"filter"`
						Adaptativebitrate       bool `json:"adaptativeBitrate"`
						Onlyplaywithsubscribers bool `json:"onlyPlayWithSubscribers"`
						Networkcache            int  `json:"networkCache"`
					} `json:"mediaOptions"`
				} `json:"publishers"`
				Subscribers []interface{} `json:"subscribers"`
			} `json:"content"`
		} `json:"connections"`
		Recording bool `json:"recording"`
	} `json:"content"`
}


func GetSessionActive(js []byte) int {
	var session1 ViduSession
	var json = jsoniter.ConfigCompatibleWithStandardLibrary
	json.Unmarshal(js, &session1)
	return session1.Numberofelements
}

func GetConnectionActive(js []byte) int {
	var session1 ViduSession
	var connections int = 0
	var json = jsoniter.ConfigCompatibleWithStandardLibrary
	json.Unmarshal(js, &session1)
	for i := 0; i < session1.Numberofelements; i++ {
		connections += session1.Content[i].Connections.Numberofelements
	}
	return connections
}
```

** ตัวอย่าง code ดูได้จาก https://github.com/neverlock/openvidu-exporter/
