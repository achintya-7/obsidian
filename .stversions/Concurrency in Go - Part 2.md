## Channels
In Go language, a channel is a medium through which a goroutine communicates with another goroutine and this communication is lock-free. Or in other words, a channel is a technique which allows to let one goroutine to send data to another goroutine.

Lets make a simple program with channel 
```
func main() {
    now := time.Now()
 

    defer func () {

        fmt.Println(time.Since(now))

    }()

  

    channel := make(chan string)

  

    go count("Hello", channel)

  

    fmt.Println(<-channel)

}

  

func count(str string, channel chan string) {

    time.Sleep(time.Second * 1)

    channel <- str

}
```