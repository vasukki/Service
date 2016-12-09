package main

import (
	"log"
	"net"
	"os"
	"github.com/kardianos/service"
	"github.com/kataras/iris"
	"fmt"
	"io/ioutil"
	"strconv"
	"strings"
	"time"
)

var logger service.Logger

type program struct{}

func (p *program) Start(s service.Service) error {
	go p.run()
	return nil
}

func getCPUSample() (idle, total uint64)  {
	contents, err := ioutil.ReadFile("/proc/stat")

	if err != nil{
		return
	}

	lines := strings.Split(string(contents), "\n")

	for _, line := range(lines) {
		fields := strings.Fields(line)

		if fields[0]== "cpu" {
			numFields := len(fields)

			for i := 1; i < numFields; i++{
				val, err := strconv.ParseUint(fields[i], 10, 64)

				if err != nil {
					fmt.Println("Error:", i, fields[i], err)
				}

				total += val
				if i == 4 {
					idle = val
				}
			}
			return
		}
	}
	return
}

func (p *program) run()  {
	iris.Get("/component=ip", func (ctx *iris.Context)  {
		addrs, err := net.InterfaceAddrs()

		if err != nil {
			os.Stderr.WriteString("Oops: " + err.Error() + "\n")
			os.Exit(1)
		}

		for _, a := range addrs{

			if ipnet, ok := a.(*net.IPNet); ok && !ipnet.IP.IsLoopback() {

				if ipnet.IP.To4() != nil{
					os.Stdout.WriteString(ipnet.IP.String() + "\n")
					ctx.Write("The IP address is !")
					ctx.Write(ipnet.IP.String() + "\n")
				}
		 }
		}
	})

iris.Get("/component=txt", func (ctx *iris.Context)  {
	ctx.Write("Hello, %s","This program monitors the Performance")
})

iris.Get("/component=cpuUsage", func(ctx *iris.Context){
	idle0, total0 := getCPUSample()
	time.Sleep(3 * time.Second)
	idle1, total1 := getCPUSample()

	idleTicks := float64(idle1 - idle0)
	totalTicks := float64(total1 - total0)
	cpuUsage := 100 * (totalTicks -idleTicks) / totalTicks

	ctx.Write("CPU usage is %f%% [busy: %f, total: f]\n", cpuUsage, totalTicks-idleTicks, totalTicks)
})
iris.Listen(":8072")
}


func (p *program) Stop(s service.Service) error {
	return nil
}

func main() {
	svcConfig := &service.Config{
		Name:"GoServiceTest",
		DisplayName:"Go Service Test",
		Description: "This is a test Go service.",
	}


prg := &program{}
s, err := service.New(prg, svcConfig)

if err != nil {
	log.Fatal(err)
}

logger, err = s.Logger(nil)
if err != nil {
	log.Fatal(err)
}

err = s.Run()
if err !=nil {
	logger.Error(err)
}
}
