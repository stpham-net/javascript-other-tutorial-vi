# Node.js Child Processes: Everything you need to know

## How to use spawn(), exec(), execFile(), and fork()

![](https://cdn-images-1.medium.com/max/2000/1*I56pPhzO1VQw8SIsv8wYNA.png)

*Ảnh chụp màn hình được chụp từ khóa học Pluralsight của tôi -- Node.js nâng cao*

Đơn luồng (single-threaded), không chặn (non-blocking) thực hiện trong Node.js hoạt động tuyệt vời cho một process (single process). Nhưng cuối cùng, một process trong một CPU sẽ không đủ để xử lý khối lượng công việc ngày càng tăng của ứng dụng của bạn.

Cho dù máy chủ của bạn có mạnh đến đâu, một luồng chỉ có thể hỗ trợ tải hạn chế (limited load).

Việc Node.js chạy trong một luồng không có nghĩa là chúng ta không thể tận dụng multiple processes và tất nhiên, multiple machines cũng vậy.

Sử dụng multiple processes là cách tốt nhất để mở rộng (scale) ứng dụng Node. Node.js được thiết kế để xây dựng các ứng dụng phân tán (distributed applications) với nhiều nút (nodes). Đây là lý do tại sao nó có tên *Nút (Node)*. Khả năng mở rộng được đưa vào nền tảng và đó không phải là điều bạn bắt đầu nghĩ đến sau này trong vòng đời của một ứng dụng.

> Bài viết này là một phần của [khóa học Pluralsight của tôi về Node.js](https://www.pluralsight.com/cifts/nodejs-advified). Tôi bao gồm nội dung tương tự ở định dạng video ở đó.

Xin lưu ý rằng bạn sẽ cần hiểu rõ về các Node.js *events* và *streams* trước khi bạn đọc bài viết này. Nếu bạn chưa biết, tôi khuyên bạn nên đọc hai bài viết này trước khi bạn đọc bài viết này:

[Tìm hiểu kiến trúc hướng sự kiện của Node.js](https://medium.freecodecamp.com/understanding-node-js-event-driven-architecture-223292fcbc2d)

[Node.js Streams: Mọi thứ bạn cần biết](https://medium.freecodecamp.com/node-js-streams-everything-you-need-to-know-c9141306be93)

### The Child Processes Module

Chúng ta có thể dễ dàng quay (spin) một child process bằng cách sử dụng mô đun `child_ process` của Node và các child processes đó có thể dễ dàng giao tiếp với nhau bằng một hệ thống nhắn tin (messaging system).

Mô-đun `child_ process` cho phép chúng ta truy cập các chức năng của Hệ điều hành (Operating System functionalities) bằng cách chạy bất kỳ lệnh hệ thống (system command) nào bên trong một child process, well.

Chúng ta có thể kiểm soát (control) luồng đầu vào (input stream) của child process đó và listen luồng đầu ra (output stream) của nó. Chúng ta cũng có thể kiểm soát các đối số được truyền bên dưới OS command và chúng ta có thể làm bất cứ điều gì chúng ta muốn với đầu ra của lệnh đó. Ví dụ, chúng ta có thể đặt đầu ra của một lệnh làm đầu vào cho một lệnh khác (giống như chúng ta làm trong Linux) vì tất cả các đầu vào và đầu ra của các lệnh này có thể được trình bày cho chúng ta bằng cách sử dụng [Node.js streams](https://medium.freecodecamp.com/node-js-streams-everything-you-need-to-know-c9141306be93).

* Lưu ý rằng các ví dụ tôi sẽ sử dụng trong bài viết này đều dựa trên Linux. Trên Windows, bạn cần chuyển đổi các lệnh tôi sử dụng với các lựa chọn thay thế dành cho Windows.*

Có bốn cách khác nhau để tạo một child process trong Node: `spawn()`, `fork()`, `exec()`, and `execFile()`.

Chúng ta sẽ thấy sự khác biệt giữa bốn chức năng này và khi nào nên sử dụng mỗi chức năng.

#### Spawned Child Processes

Hàm `spawn` khởi chạy một command trong một process mới và chúng ta có thể sử dụng nó để truyền cho command đó bất kỳ đối số nào. Ví dụ, đây là mã để sinh ra một process mới sẽ thực thi lệnh `pwd`.

```js
const { spawn } = require('child_process');

const child = spawn('pwd');
```

Chúng ta chỉ cần destructure the `spawn` function ra khỏi `child_process` module và thực thi nó với OS command làm đối số đầu tiên.

Kết quả của việc thực thi hàm `spawn` (đối tượng `child` ở trên) là một phiên bản `ChildProcess`, thực hiện [EventEmitter API](https://medium.freecodecamp.com/understanding-node-js-event-driven-architecture-223292fcbc2d). Điều này có nghĩa là chúng ta có thể đăng ký handlers cho các events trên child object này trực tiếp. Ví dụ, chúng ta có thể làm gì đó khi child process thoát ra bằng cách đăng ký một handler cho `exit` event:

```js
child.on('exit', function (code, signal) {
  console.log('child process exited with ' +
              `code ${code} and signal ${signal}`);
});
```

The handler ở trên cung cấp cho chúng ta exit `code` cho the child process và `signal`, nếu có, được sử dụng để chấm dứt the child process. `signal` variable này là null khi the child process exits bình thường.

Các events khác mà chúng ta có thể đăng ký handlers với the `ChildProcess` instances là `disconnect`, `error`, `close`, và `message`.

- The `disconnect` event được emitted khi parent process gọi thủ công `child.disconnect` function.
- The `error` event được emitted nếu the process không thể spawned hoặc killed.
- The `close` event được emitted khi các `stdio` streams của một child process bị đóng lại.
- The `message` event là quan trọng nhất. Nó được emitted khi the child process sử dụng hàm `process.send()` để gửi messages. Đây là cách các parent/child processes có thể giao tiếp với nhau. Chúng ta sẽ thấy một ví dụ về điều này dưới đây.

Mọi child process cũng nhận được ba `stdio` streams tiêu chuẩn, mà chúng ta có thể truy cập bằng cách sử dụng `child.stdin`, `child.stdout`, và `child.stderr`.

Khi các streams đó bị đóng, the child process đang sử dụng chúng sẽ phát ra `close` event. `close` event này khác với the `exit` event vì nhiều child processes có thể chia sẻ cùng một `stdio` streams và vì vậy một child process thoát ra không có nghĩa là các streams đã bị đóng.

Vì tất cả các streams là các event emitters, chúng ta có thể listen các events khác nhau trên các `stdio` streams được gắn vào mọi child process. Không giống như trong một process thông thường, trong một child process, các `stdout`/`stderr` streams là các readable streams trong khi `stdin` stream là một writable stream. Điều này về cơ bản là ngược lại với các kiểu được tìm thấy trong một main process. Các events chúng ta có thể sử dụng cho các streams đó là các events tiêu chuẩn. Quan trọng nhất, trên các readable streams, chúng ta có thể listen to the `data` event, mà sẽ có đầu ra của command hoặc bất kỳ error nào gặp phải khi thực thi command:

```js
child.stdout.on('data', (data) => {
  console.log(`child stdout:\n${data}`);
});

child.stderr.on('data', (data) => {
  console.error(`child stderr:\n${data}`);
});
```

Hai handlers ở trên sẽ log cả hai trường hợp vào the main process `stdout` và `stderr`. Khi chúng ta thực thi hàm `spawn` ở trên, đầu ra của `pwd` command sẽ được in (printed) và the child process thoát ra (exits) với mã `0`, có nghĩa là không có lỗi xảy ra.

Chúng ta có thể truyền các đối số cho command được thực thi bởi hàm `spawn` bằng cách sử dụng đối số thứ hai của hàm `spawn`, là một mảng của tất cả các đối số được truyền cho the command. Ví dụ, để thực thi lệnh `find` trên thư mục hiện tại với một đối số `-type f` (chỉ liệt kê các tệp), chúng ta có thể làm:

```js
const child = spawn('find', ['.', '-type', 'f']);
```

Nếu xảy ra lỗi trong quá trình thực thi the command, ví dụ, nếu chúng ta tìm thấy đích không hợp lệ ở trên, the `child.stderr` `data` event handler sẽ được kích hoạt và the `exit` event handler sẽ báo cáo một exit code of `1`, điều đó biểu thị rằng đã xảy ra lỗi. Các giá trị lỗi thực sự phụ thuộc vào host OS và kiểu của lỗi.

A child process `stdin` là một writable stream. Chúng ta có thể sử dụng nó để send a command some input. Giống như bất kỳ writable stream nào, cách dễ nhất để sử dụng nó là sử dụng `pipe` function. Chúng ta chỉ đơn giản pipe một readable stream tới một writable stream. Vì the main process `stdin` là một readable stream, nên chúng ta có thể pipe luồng đó thành một child process `stdin` stream. Ví dụ:

```js
const { spawn } = require('child_process');

const child = spawn('wc');

process.stdin.pipe(child.stdin)

child.stdout.on('data', (data) => {
  console.log(`child stdout:\n${data}`);
});
```

Trong ví dụ trên, the child process gọi ra (invokes) the `wc` command, đếm các dòng, từ và ký tự trong Linux. Sau đó, chúng ta pipe the main process `stdin` (mà là một readable stream) tới child process `stdin` (mà là một writable stream). Kết quả của sự kết hợp này là chúng ta có được một chế độ đầu vào tiêu chuẩn nơi chúng ta có thể nhập một cái gì đó và khi chúng ta nhấn `Ctrl+D`, những gì chúng ta đã nhập sẽ được sử dụng làm đầu vào của `wc` command.

![](https://cdn-images-1.medium.com/max/1000/1*s9dQY9GdgkkIf9zC1BL6Bg.gif)

*Ảnh chụp màn hình được chụp từ khóa học Pluralsight của tôi -- Node.js nâng cao*

Chúng ta cũng có thể pipe the standard input/output of multiple processes lên nhau, giống như chúng ta có thể làm với các lệnh Linux. Ví dụ, chúng ta có thể pipe the `stdout` của `find` command tới the stdin of the `wc` command để đếm tất cả các tệp trong thư mục hiện tại:

```js
const { spawn } = require('child_process');

const find = spawn('find', ['.', '-type', 'f']);
const wc = spawn('wc', ['-l']);

find.stdout.pipe(wc.stdin);

wc.stdout.on('data', (data) => {
  console.log(`Number of files ${data}`);
});
```

Tôi đã thêm đối số `-l` vào `wc` command để làm cho nó chỉ đếm các dòng (count only the lines). Khi được thực thi, đoạn mã trên sẽ xuất ra một count của tất cả các tệp trong tất cả các thư mục dưới (under) tệp hiện tại.

#### Shell Syntax and the exec function

Theo mặc định, hàm `spawn` không tạo một *shell* để thực thi lệnh chúng ta truyền vào nó. Điều này làm cho nó hiệu quả hơn một chút so với hàm `exec`, nó tạo ra một shell. Hàm `exec` có một sự khác biệt lớn khác. Nó *đệm (buffers)* đầu ra được tạo của lệnh và passes toàn bộ giá trị đầu ra tới một callback function (thay vì sử dụng các streams, đó là những gì `spawn` làm).

Đây là ví dụ `find | wc` được triển khai với hàm `exec`.

```js
const { exec } = require('child_process');

exec('find . -type f | wc -l', (err, stdout, stderr) => {
  if (err) {
    console.error(`exec error: ${err}`);
    return;
  }

  console.log(`Number of files ${stdout}`);
});
```

Vì hàm `exec` sử dụng shell để thực thi lệnh (command), nên chúng ta có thể sử dụng *shell syntax* trực tiếp tại đây bằng cách sử dụng tính năng shell *pipe*.

Lưu ý rằng việc sử dụng shell syntax có [rủi ro bảo mật](https://blog.liftsecurity.io/2014/08/19/Avoid-Command-Injection-Node.js/) nếu bạn đang thực hiện bất kỳ loại dynamic input nào được cung cấp từ bên ngoài. Người dùng có thể chỉ cần thực hiện một command injection attack bằng cách sử dụng các shell syntax characters như `;` và `$` (ví dụ: `command + '; rm -rf ~'`)

Hàm `exec` buffers the output và chuyển nó đến callback function (đối số thứ hai cho `exec`) như là đối số `stdout` ở đó. Đối số `stdout` này là đầu ra của lệnh mà chúng ta muốn in ra.

Hàm `exec` là một lựa chọn tốt nếu bạn cần sử dụng shell syntax và nếu kích thước của dữ liệu dự kiến từ lệnh là nhỏ. (Hãy nhớ, `exec` sẽ đệm (buffer) toàn bộ dữ liệu trong bộ nhớ trước khi trả lại.)

Hàm `spawn` là một lựa chọn tốt hơn nhiều khi kích thước của dữ liệu được mong đợi từ lệnh lớn, bởi vì dữ liệu đó sẽ được streamed với các standard IO objects.

Chúng ta có thể làm cho the spawned child process kế thừa các the standard IO objects của parents nếu chúng ta muốn, nhưng, quan trọng hơn, chúng ta cũng có thể làm cho hàm `spawn` sử dụng shell syntax. Đây là cùng một `find | wc` command được triển khai với hàm `spawn`:

```js
const child = spawn('find . -type f | wc -l', {
  stdio: 'inherit',
  shell: true
});
```

Do tùy chọn `stdio: 'inherit'` ở trên, khi chúng ta thực thi mã, the child process kế thừa main process `stdin`, `stdout`, và `stderr`. Điều này làm cho các the child process data events handlers được kích hoạt trên main `process.stdout` stream, làm cho script xuất kết quả ngay lập tức.

Do tùy chọn `shell: true` ở trên, chúng ta có thể sử dụng the shell syntax trong lệnh đã truyền (passed command), giống như chúng ta đã làm với `exec`. Nhưng với mã này, chúng ta vẫn có được lợi thế của việc streaming of data mà hàm `spawn` mang lại cho chúng ta. *Đây thực sự là tốt nhất của cả hai thế giới.*

Có một vài tùy chọn tốt khác mà chúng ta có thể sử dụng trong đối số cuối cùng cho các the `child_process` functions bên cạnh `shell` và `stdio`. Ví dụ, chúng ta có thể sử dụng tùy chọn `cwd` để thay đổi thư mục làm việc của script. Ví dụ, đây là ví dụ the same count-all-files example được thực hiện với hàm `spawn` sử dụng một shell và với một thư mục làm việc được đặt tới my Downloads folder. Tùy chọn `cwd` ở đây sẽ làm cho script đếm tất cả các tệp tôi có trong `~/Downloads`:

```js
const child = spawn('find . -type f | wc -l', {
  stdio: 'inherit',
  shell: true,
  cwd: '/Users/samer/Downloads'
});
```

Một tùy chọn khác mà chúng ta có thể sử dụng là tùy chọn `env` để chỉ định các environment variables sẽ hiển thị cho the new child process. Mặc định cho tùy chọn này là `process.env`, cho phép mọi lệnh truy cập vào process environment hiện tại. Nếu chúng ta muốn ghi đè hành vi đó, chúng ta có thể chuyển một đối tượng trống dưới dạng tùy chọn `env` hoặc các giá trị mới ở đó để được coi là environment variables duy nhất:

```js
const child = spawn('echo $ANSWER', {
  stdio: 'inherit',
  shell: true,
 env: { ANSWER: 42 },
 });
```

Lệnh echo ở trên không có quyền truy cập vào các parent process's environment variables. Ví dụ, nó không thể truy cập `$HOME`, nhưng nó có thể truy cập `$ANSWER` vì nó được passed như là một custom environment variable thông qua tùy chọn `env`.

Một tùy chọn child process quan trọng cuối cùng cần giải thích ở đây là tùy chọn `detached`, làm cho the child process chạy độc lập với parent process của nó.

Giả sử chúng ta có một tệp `timer.js` giữ cho event loop bận rộn:

```js
setTimeout(() => {
  // keep the event loop busy
}, 20000);
```

Chúng ta có thể thực thi nó trong background bằng cách sử dụng tùy chọn `detached`:

```js
const { spawn } = require('child_process');

const child = spawn('node', ['timer.js'], {
 detached: true,
  stdio: 'ignore'
});

child.unref();
```

Hành vi chính xác của các detached child processes phụ thuộc vào OS. Trên Windows, the detached child process sẽ có console window riêng trong khi trên Linux, the detached child process sẽ được đưa lên làm thủ lĩnh (leader) của một new process group và session.

Nếu hàm `unref` được gọi trên the detached process, the parent process có thể exit độc lập với the child. Điều này có thể hữu ích nếu the child đang thực hiện một long-running process, nhưng để giữ cho nó chạy trong background, các cấu hình `stdio` của the child cũng phải độc lập với the parent.

Ví dụ trên sẽ chạy node script (`timer.js`) trong background bằng cách tách ra và cũng bỏ qua its parent `stdio` file descriptors để the parent có thể chấm dứt (terminate) trong khi the child tiếp tục chạy trong background.

![](https://cdn-images-1.medium.com/max/1000/1*WhvMs8zv-WS6v7nDXmDUzw.gif)

*Ảnh chụp màn hình được chụp từ khóa học Pluralsight của tôi -- Node.js nâng cao*

#### The execFile function

Nếu bạn cần thực thi một tệp mà không sử dụng một shell, hàm `execFile` là thứ bạn cần. Nó hoạt động chính xác như hàm `exec`, nhưng không sử dụng một shell, điều này làm cho nó hiệu quả hơn một chút. Trên Windows, một số tệp không thể tự thực thi, như các tệp `.bat` hoặc `.cmd`. Các tệp đó không thể thực thi bằng `execFile` và `exec` hoặc `spawn` với shell được đặt thành true là bắt buộc để thực thi chúng.

#### The *Sync function

Các hàm `spawn`, `exec` và `execFile` từ mô đun `child_process` cũng có các phiên bản chặn đồng bộ (synchronous blocking) mà sẽ đợi cho đến khi the child process exits.

```js
const {
  spawnSync,
  execSync,
  execFileSync,
} = require('child_process');
```

Các phiên bản đồng bộ này có khả năng hữu ích khi cố gắng đơn giản hóa các tác vụ kịch bản (scripting tasks) hoặc bất kỳ startup processing tasks nào, nhưng chúng nên được tránh bằng cách khác.

#### The fork() function

The `fork` function là một biến thể của hàm `spawn` để spawning các node processes. Sự khác biệt lớn nhất giữa `spawn` và `fork` là đó là một communication channel được thiết lập cho the child process khi sử dụng `fork`, vì vậy chúng ta có thể sử dụng hàm `send` trên the forked process cùng với chính the global `process` object để trao đổi messages giữa the parent và forked processes. Chúng ta thực hiện điều này thông qua the `EventEmitter` module interface. Đây là một ví dụ:

The parent file, `parent.js`:

```js
const { fork } = require('child_process');

const forked = fork('child.js');

forked.on('message', (msg) => {
  console.log('Message from child', msg);
});

forked.send({ hello: 'world' });
```

The child file, `child.js`:

```js
process.on('message', (msg) => {
  console.log('Message from parent:', msg);
});

let counter = 0;

setInterval(() => {
  process.send({ counter: counter++ });
}, 1000);
```

Trong parent file ở trên, chúng ta fork `child.js` (sẽ thực thi the file bằng `node` command) và sau đó chúng ta listen the `message` event. The `message` event sẽ được phát ra (emitted) bất cứ khi nào the child sử dụng `process.send`, chúng ta đang làm mỗi giây.

Để pass down messages từ the parent tới the child, chúng ta có thể thực thi hàm `send` trên chính the forked object, và sau đó, trong the child script, chúng ta có thể listen to the `message` event trên the global `process` object.

Khi thực thi the `parent.js` file ở trên, trước tiên nó sẽ gửi xuống the `{ hello: 'world' }` object sẽ được in (printed) bởi the forked child process và sau đó the forked child process sẽ gửi một incremented counter value mỗi giây được in (printed) bởi the parent process.

![](https://cdn-images-1.medium.com/max/1000/1*GOIOTAZTcn40qZ3JwgsrNA.gif)

*Ảnh chụp màn hình được chụp từ khóa học Pluralsight của tôi -- Node.js nâng cao*

Chúng ta hãy làm một ví dụ thực tế hơn về hàm `fork`.

Giả sử chúng ta có một http server xử lý hai endpoints. Một trong những endpoints này (`/compute` bên dưới) tính toán rất nhiều và sẽ mất vài giây để hoàn thành. Chúng ta có thể sử dụng một long for loop để mô phỏng rằng:

```js
const http = require('http');

const longComputation = () => {
  let sum = 0;
  for (let i = 0; i < 1e9; i++) {
    sum += i;
  };
  return sum;
};

const server = http.createServer();

server.on('request', (req, res) => {
  if (req.url === '/compute') {
    const sum = longComputation();
    return res.end(`Sum is ${sum}`);
  } else {
    res.end('Ok')
  }
});

server.listen(3000);
```

This program có một vấn đề lớn; khi the `/compute` endpoint được requested, the server sẽ không thể xử lý bất kỳ requests nào khác vì the event loop đang bận với hoạt động the long for loop.

Có một số cách chúng ta có thể giải quyết vấn đề này tùy thuộc vào bản chất của độ dài hoạt động (the long operation) nhưng một giải pháp hiệu quả cho tất cả các hoạt động là chỉ cần chuyển hoạt động tính toán (the computational operation) sang process khác bằng cách sử dụng `fork`.

Trước tiên chúng ta di chuyển toàn bộ hàm `longComputation` vào tệp riêng của nó và làm cho nó gọi ra hàm đó khi được hướng dẫn qua một message từ the main process:

Trong tệp `compute.js` mới:

```js
const longComputation = () => {
  let sum = 0;
  for (let i = 0; i < 1e9; i++) {
    sum += i;
  };
  return sum;
};

process.on('message', (msg) => {
  const sum = longComputation();
  process.send(sum);
});
```

Bây giờ, thay vì thực hiện the long operation trong the main process event loop, chúng ta có thể `fork` tệp `compute.js` và sử dụng the messages interface để giao messages giữa the server và the forked process.

```js
const http = require('http');
const { fork } = require('child_process');

const server = http.createServer();

server.on('request', (req, res) => {
  if (req.url === '/compute') {
    const compute = fork('compute.js');
    compute.send('start');
    compute.on('message', sum => {
      res.end(`Sum is ${sum}`);
    });
  } else {
    res.end('Ok')
  }
});

server.listen(3000);
```

Khi một request to `/compute` xảy ra với mã ở trên, chúng ta chỉ cần gửi một message đến the forked process để bắt đầu thực thi the long operation. The main process's event loop sẽ không bị chặn.

Khi the forked process thực hiện xong với long operation đó, nó có thể gửi kết quả của nó trở lại the parent process bằng cách sử dụng `process.send`.

Trong the parent process, chúng ta listen to the `message` event trên chính the forked child process. Khi chúng ta nhận được event đó, chúng ta sẽ có một giá trị `sum` sẵn sàng để chúng ta gửi cho người dùng yêu cầu (requesting user) qua http.

Tất nhiên, đoạn mã trên bị giới hạn bởi số lượng processes chúng ta có thể fork, nhưng khi chúng ta thực thi nó và request the long computation endpoint qua http, the main server hoàn toàn không bị chặn và có thể take further requests.

Node's `cluster` module, là chủ đề của bài viết tiếp theo của tôi, dựa trên ý tưởng này của child process forking và load balancing các the requests thông qua many forks mà chúng ta có thể tạo trên bất kỳ hệ thống nào.

Đó là tất cả những gì tôi có cho chủ đề này. Cảm ơn vì đã đọc! Cho đến lần sau!

* * * * *

Learning React or Node? Checkout my books:

- [Learn React.js by Building Games](http://amzn.to/2peYJZj)
- [Node.js Beyond the Basics](http://amzn.to/2FYfYru)
