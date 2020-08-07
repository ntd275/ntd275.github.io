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

```golang
//n là số goroutine đồng thời
//m là số lần truy cập và biến dùng chung của 1 goroutine
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
```golang
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
```golang
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
- ```Mutex``` có tốc độ nhanh thứ hai khoảng 134.4ms trên một lần chạy
- channel có tốc độ chậm nhất khoảng 496.4ms trên một lần chạy