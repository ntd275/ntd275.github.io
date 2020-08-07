---
layout: post
title:  "So sánh hiệu năng các phương pháp tránh xung đột khi truy cập biến dùng chung trong Go"
date:   2020-08-06 11:18:20 +0700
categories: golang
---
# So sánh hiệu năng các phương pháp tránh xung đột khi truy cập biến dùng chung trong Go
Khi lập trình song song, các luồng hoạt động cùng lúc có thể phải truy cập vào các biến dùng chung. Ta cần đảm bảo tại một thời điểm chỉ có duy nhất một luông truy cập vào biến đó nếu không sẽ xảy ra xung đột, chương trình sẽ trả về kết quả sai   

Một ví dụ đơn giản đó là bộ đếm nếu hai luồng cùng truy cập và thay tăng giá trị biến đếm lên 1 đơn vị thì sao
<table>
    <tr>
        <th> Thread 1</th>
        <th> Thread 2</th>
        <th> Counter </th>
    </tr>
    <tr>
        <td> </td>
        <td> </td>
        <td>0</td>
    </tr>
    <tr>
        <td>Đọc giá trị</td>
        <td> </td>
        <td>0</td>
    </tr>
    <tr>
        <td>Tăng giá trị lên 1</td>
        <td>Đọc giá trị</td>
        <td>0</td>
    </tr>
    <tr>
        <td>Viết giá trị</td>
        <td>Tăng giá trị lên 1</td>
        <td>1</td>
    </tr>
    <tr>
        <td></td>
        <td>Viết giá trị</td>
        <td>1</td>
    </tr>
</table>

Ở đây hai luồng cùng tăng giá trị của counter lên 1 nhưng sau khi thực hiện counter không được tăng thành 2 mà chỉ là 1

Trong Go để giải quyết vấn đề này có nhiều giải pháp và mình sẽ đề cập đến 3 giải pháp phổ biến được dùng
- Atomic
- Mutex
- Channel
## Cách test hiệu năng
Để test hiệu năng mình sẽ làm một nhiệm vụ đơn giản đó là tăng giá trị của một biến dùng chung bằng nhiều goroutine và so sánh thời gian chạy của các phương pháp sử dụng bằng ```go test```   
Nhiệm vụ này có thể cài đặt như sau:   
main.go

```go
package main

import (
	"fmt"
	"os"
	"strconv"
	"sync"
	"time"
)

//n là số goroutine đồng thời
//m là số lần truy cập và biến dùng chung của 1 goroutine
func test(n int, m int) int64 {
	var sum int64
	wg := sync.WaitGroup{}
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			for i := 0; i < m; i++ {
				sum++
			}
			wg.Done()
		}()
	}
	wg.Wait()
	return sum
}

func main() {
	n, _ := strconv.Atoi(os.Args[1])
	m, _ := strconv.Atoi(os.Args[1])
	start := time.Now()
	fmt.Printf("sum = %d\n", test(n, m))
	t := time.Since(start)
	fmt.Printf("time : %v", t)
}
```

Ở đây biến ```sum``` là biến dùng chung của các goroutine cần phải tránh truy cập đồng thời khi chạy thử bằng lệnh   


---
Sử dụng ```atomic```:    
```go
//n là số goroutine đồng thời
//m là số lần truy cập và biến dùng chung của 1 goroutine
func test(n int, m int) int64 {
	var sum int64
	wg := sync.WaitGroup{}
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			for i := 0; i < m; i++ {
				atomic.AddInt64(&sum, 1)
			}
			wg.Done()
		}()
	}
	wg.Wait()
	return sum
}
```
Sử dụng hàm ```atomic.AddInt64(&sum, 1)``` để tăng giá trị của sum. Hàm này sử dụng lệnh nguyên tử cấp thấp của CPU nên đảm bảo trong lúc nó thực hiện không có bất kỳ lệnh nào khác cùng truy cập vào biến sum.   

---
Sử dụng ```Mutex```:
```go
//n là số goroutine đồng thời
//m là số lần truy cập và biến dùng chung của 1 goroutine
func test(n int, m int) int64 {
	var sum int64
	wg := sync.WaitGroup{}
	mu := sync.Mutex{}
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			for i := 0; i < m; i++ {
				mu.Lock()
				sum++
				mu.Unlock()
			}
			wg.Done()
		}()
	}
	wg.Wait()
	return sum
}
```
Với ```Mutex``` ta sử dụng cơ chế khóa. ```mu.Lock()``` sẽ phải đợi đến khi nào trạng thái là không khóa thì sẽ khóa lại và thực hiện câu lệnh ```sum++``` sau khi xong ```mu.Unlock()``` sẽ mở khóa để cho phép các luồng khác có thể khóa và truy cập vào ```sum```. ```Mutex``` được cài đặt dựa trên ```atomic.CompareAndSwapInt32()``` là câu lệnh nguyên tử đảm bảo sẽ không có hai luồng đồng thời cùng thực hiện khóa.

---
Sử dụng Channel:   
```golang
func test(n int, m int) int64 {
	var sum int64
	wg := sync.WaitGroup{}
	ch := make(chan struct{}, 1)
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			for i := 0; i < m; i++ {
				ch <- struct{}{}
				sum++
				<-ch
			}
			wg.Done()
		}()
	}
	wg.Wait()
	return sum
}
```
Sử dụng tính chất của buffer channel trong Go ta sử dụng nó như một khóa ```ch := make(chan struct{}, 1)``` tạo thành channel có kiểu là chan struct{} và buffer là 1, do channel này không cần truyền dữ liệu gì mà chỉ là truyền một tín hiệu nên ta sử dụng kiểu ```chan struct{}```. Trước khi thay đổi ```sum``` ta cần gửi 1 tín hiệu vào channel do channel có buffer là 1 nên chỉ có một goroutine có thể gửi vào tại một thời điểm việc này tương tự khóa. Sau khi thay đổi ```sum``` goroutine sẽ nhận lại tín hiệu để channel rỗng lúc đó các goroutine khác mới có thể truy cập vào ```sum``` việc này tương tự mở khóa.

---
Hàm dùng để test:   
main_test.go
```golang
package main

import "testing"

func Benchmark(b *testing.B) {
	for i := 0; i < b.N; i++ {
		test(1000, 1000)
	}
}
```
Ở đây ta cho 1000 goroutine chạy cùng lúc và mỗi goroutine sẽ tăng sum lên 1000 lần

----
## Kết quả
Sử dụng ```go test -bench=. -benchtime=20s``` để benchmark ta có kết quả như sau   
Khi dùng ```atomic```:
```
D:\ducnt59\Go Project\test concurency\atomic>go test -bench= -benchtime=20s
goos: windows
goarch: amd64
pkg: atomic
BenchmarkAtomic-4           1377          17590694 ns/op
PASS
ok      atomic  26.086s
```
Khi dùng ```Mutex```:
```
D:\ducnt59\Go Project\test concurency\mutex>go test -bench=. -benchtime=20s
goos: windows
goarch: amd64
pkg: mutex
BenchmarkAtomic-4            183         134357411 ns/op
PASS
ok      mutex   37.825s
```

Khi dùng channel:
```
D:\ducnt59\Go Project\test concurency\channel>go test -bench=. -benchtime=20s
goos: windows
goarch: amd64
pkg: channel
BenchmarkAtomic-4             46         496397957 ns/op
PASS
ok      channel 42.383s
```
### Nhận xét
- ```atomic``` có tốc độ thực hiện nhanh nhất trung bình khoảng 17.6ms trên một lần chạy
- ```Mutex``` có tốc độ nhanh thứ hai khoảng 134.4ms trên một lần chạy gấp hơn 7 lần ```atomic```
- channel có tốc độ chậm nhất khoảng 496.4ms trên một lần chạy gấp hơn 28 lần ```atomic```

### Lý do vì sao có sự chênh lệch như vậy?   

Ta cần tìm hiểu công nghệ phía sau các phương pháp này:   
- ```atomic``` sử dụng câu lệnh nguyên tử cấp thấp do CPU cung cấp vì vậy nó rất nhanh không chậm hơn so với việc thực hiện thông thường quá nhiều. Câu lệnh nguyên tử không phải là chỉ thực hiện một lệnh trên CPU mà bản chất là khi nó thực hiện không có bất kì câu lệnh nào có thể truy cập đồng thời vào tài nguyên. Từ góc nhìn của các luồng khác có thể coi đó như một lệnh nguyên tử không thể chia nhỏ và xen vào giữa khi nó thực hiện. Để làm được điều này cách đơn giản nhất là CPU sử dụng tín hiệu LOCK trên bus để đảm bảo không có bộ sử lý nào khác truy cập vào bộ nhớ cùng lúc.
- ```Mutex``` được xây dựng trên ```atomic``` dùng một biến làm khóa
dùng lệnh nguyên tử ```atomic.CompareAndSwapInt32()``` để thay đổi trạng thái khóa và mở khóa của biến này trong khi biến ở trạng thái khóa thì chỉ có duy nhất luồng thực hiện khóa biến đó truy cập vào các tài nguyên dùng chung và cũng chỉ luồng đó mới mở khóa khi thực hiện xong để các luồng khác có thể khóa và truy cập. Khi có nhiều goroutine cùng truy cập thì CPU dành phần lớn thời gian để kiểm tra khóa. Vì các lý do trên nên ```Mutex``` chậm hơn ```atomic``` như vậy.
- Channel là một struct giúp chúng ta gửi nhận thông tin giữa các goroutine. Để đảm bảo không tương tranh trước khi goroutine gửi vào channel sẽ bị khóa sau đó nó sẽ kiểm tra xem bộ đệm có trống hay không nếu có sẽ đưa dữ liệu vào bộ đệm, nếu có goroutine trong hàng đợi nhận nó sẽ đánh thức goroutine này để nhận dữ liệu sau đó mở khóa channel rồi thực hiệp tiếp công việc, nếu bộ đệm đầy goroutine sẽ bị đưa vào hàng đợi và chuyển sang trạng thái bị block rồi mở khóa channel. Khi nhận thì ngược lại trước khi nhận goroutine cũng sẽ khóa channel rồi kiểm tra bộ đệm xem có dữ liệu hay không nếu có sẽ lấy dữ liệu và nếu hàng đợi gửi có goroutine đang chờ nó sẽ đánh thức goroutine đó để gửi dữ liệu vào channel sau đó mở khóa channel và tiếp tục công việc. Với cách cài trên thì các goroutine sẽ chỉ ở trong hàng đợi gửi và thực hiện lần lượt và sau khi xong nó lại nhận lại dữ liệu đã gửi và đánh thức goroutine tiếp theo trong hàng đợi để thực hiện tiếp. Cách này sử dụng 4 lần khóa và mở khóa cho mỗi thao tác cùng các thao tác enqueue, dequeue và goruntime cũng phải liên tục chuyển trạng thái của các goroutine nên nó chậm nhiều như vậy. Mình không khuyến khích các bạn sử dụng channel để làm khóa, nó thật ngu ngốc, hãy sử dụng đúng với mục đích channel được sinh ra là gửi nhận thông điệp giữa các goroutine.

## Nếu chỉ dùng một luồng thì sao ?
Ta có thể chỉ định chương trình chỉ dùng một luồng bằng lệnh ```runtime.GOMAXPROCS(1)```   
Khi đó hàm để test sẽ như sau:
```golang
func BenchmarkAtomic(b *testing.B) {
	runtime.GOMAXPROCS(1)
	for i := 0; i < b.N; i++ {
		test(1000, 1000)
	}
}
```
Khi dùng ```atomic```:
```
D:\ducnt59\Go Project\test concurency\atomic>go test -bench=. -benchtime=20s
goos: windows
goarch: amd64
pkg: atomic
BenchmarkAtomic-4           4039           5892637 ns/op
testing: BenchmarkAtomic-4 left GOMAXPROCS set to 1
PASS
ok      atomic  24.486s
```
Khi dùng ```Mutex```:
```
D:\ducnt59\Go Project\test concurency\mutex>go test -bench=. -benchtime=20s
goos: windows
goarch: amd64
pkg: mutex
BenchmarkMutex-4           1635          13235008 ns/op
testing: BenchmarkMutex-4 left GOMAXPROCS set to 1
PASS
ok      mutex   23.240s
```
Khi dùng channel:
```
D:\ducnt59\Go Project\test concurency\channel>go test -bench=. -benchtime=20s
goos: windows
goarch: amd64
pkg: channel
BenchmarkChannel-4           457          48501680 ns/op
testing: BenchmarkChannel-4 left GOMAXPROCS set to 1
PASS
ok      channel 27.560s
```
Thật ngạc nhiên phải không khi dùng một luồng lại nhanh hơn dùng nhiều luồng cùng lúc nhiều lần. Lý do bời vì công việc ta đang làm là không độc lập với nhau mỗi luồng cần phải đợi một luồng đang thực hiện công việc hoàn thành thì mới thực hiện cơ bản là trong một thời điểm cũng chỉ có một luồng chạy còn các luồng còn lại phải chờ. Thời gian để chuyển trạng thái của các luồng là khá lớn. Cũng như việc chỉ có một cái bút mà nhiều người dùng chung vậy, thay vì mỗi người viết một chút thì một người viết sẽ nhanh hơn vì không phải chuyển cái bút cho những người còn lại.   
Khi dùng đơn luồng thì ta có thể không cần dùng các phương pháp trên bởi trong một thời điểm goruntime chỉ cho một goroutine hoạt động nên sẽ không bị xung đột. Khi đó chương trình chạy còn nhanh hơn   
Đa luồng chưa chắc nhanh hơn đơn luồng, nên ta cần sử dụng goroutine hợp lý tránh lạm dụng chỉ dùng khi các công việc về cơ bản là song song tức là ít truy cập vào tài nguyên chung. Hoặc bạn cần thiết kế lại giải thuật thành song song hóa để tận dụng sức mạnh của đa luồng.
## Tài liệu tham khảo
[https://blog.ankuranand.com/2018/09/29/diving-deep-into-the-golang-channels/](https://blog.ankuranand.com/2018/09/29/diving-deep-into-the-golang-channels/)
[https://wiki.osdev.org/Atomic_operation](https://wiki.osdev.org/Atomic_operation)   
[https://golang.org/pkg/sync/atomic/](https://golang.org/pkg/sync/atomic/)   
[https://syslog.ravelin.com/so-just-how-fast-are-channels-anyway-4c156a407e45](https://syslog.ravelin.com/so-just-how-fast-are-channels-anyway-4c156a407e45)   
[https://golang.org/pkg/sync/](https://golang.org/pkg/sync/)
