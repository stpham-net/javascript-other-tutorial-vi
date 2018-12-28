# Hiểu kiến trúc hướng sự kiện của Node.js (Understanding Node.js Event-Driven Architecture)

Hầu hết các đối tượng của Node (Node's objects) -- như các HTTP requests, responses, and streams -- thực hiện mô đun `EventEmitter` để chúng có thể cung cấp cách phát (emit) và nghe (listen) các sự kiện (events).

![](https://cdn-images-1.medium.com/max/1000/1*74K5OhiYt7WTR0WuVGeNLQ.png)

Dạng đơn giản nhất của bản chất hướng sự kiện (event-driven nature) là callback style của một số Node.js functions phổ biến -- ví dụ: `fs.readFile`. Trong sự tương tự này, sự kiện sẽ được kích hoạt một lần (khi Node sẵn sàng gọi callback) và callback đóng vai trò xử lý sự kiện (event handler).

Trước tiên hãy khám phá hình thức cơ bản này.

#### Gọi cho tôi khi bạn đã sẵn sàng, Node! (Call me when you're ready, Node!)

Cách original (The original way) Node xử lý các sự kiện không đồng bộ là với callback. Điều này đã có từ lâu, trước khi JavaScript có hỗ trợ promises riêng và tính năng async/await.

Callbacks về cơ bản chỉ là các functions mà bạn chuyển đến các functions khác. Điều này có thể có trong JavaScript vì các functions là các first class objects.

Điều quan trọng là phải hiểu rằng các callbacks không đòi hỏi phải một cuộc gọi không đồng bộ trong mã. Một hàm có thể gọi callback cả đồng bộ và không đồng bộ.

Ví dụ: đây là hàm lưu trữ `fileSize` chấp nhận callback function `cb` và có thể gọi callback function đó một cách đồng bộ và không đồng bộ dựa trên một điều kiện:

```js
function fileSize(fileName, cb) {
  if (typeof fileName !== 'string') {
    return cb(new TypeError('argument should be string')); // Sync
  }

  fs.stat(fileName, (err, stats) => {
    if (err) { return cb(err); } // Async

    cb(null, stats.size); // Async
  });
}
```

Lưu ý rằng đây là một thực tiễn xấu dẫn đến lỗi không mong muốn. Design host functions để sử dụng callback hoặc luôn đồng bộ hoặc luôn không đồng bộ.

Chúng ta hãy khám phá một ví dụ đơn giản về asynchronous Node function điển hình được viết bằng một callback style:

```js
const readFileAsArray = function(file, cb) {
  fs.readFile(file, function(err, data) {
    if (err) {
      return cb(err);
    }

    const lines = data.toString().trim().split('\n');
    cb(null, lines);
  });
};
```

`readFileAsArray` có file path và một callback function. Nó đọc nội dung tệp, chia nó thành một mảng các dòng và gọi callback function với mảng đó.

Đây là một ví dụ sử dụng cho nó. Giả sử rằng chúng ta có tệp `numbers.txt` trong cùng thư mục có nội dung như thế này:

```
10
11
12
13
14
15
```

Nếu chúng ta có nhiệm vụ đếm các số lẻ trong tệp đó, chúng ta có thể sử dụng `readFileAsArray` để đơn giản hóa mã:

```js
readFileAsArray('./numbers.txt', (err, lines) => {
  if (err) throw err;

  const numbers = lines.map(Number);
  const oddNumbers = numbers.filter(n => n%2 === 1);
  console.log('Odd numbers count:', oddNumbers.length);
});
```

Mã này đọc nội dung số thành một mảng các strings, phân tích chúng dưới dạng số và đếm các số lẻ.

Node's callback style được sử dụng hoàn toàn ở đây. The callback có một đối số đầu tiên là lỗi `err` vô giá trị (that's nullable) và tiếp theo chúng ta chuyển callback như là đối số cuối cùng cho host function. Bạn nên luôn luôn làm điều đó trong các functions của bạn bởi vì mọi người có thể sẽ cho rằng như thế. Làm cho host function nhận được the callback như là đối số cuối cùng của nó và làm cho the callback mong đợi một error object như là đối số đầu tiên của nó.

#### JavaScript hiện đại thay thế cho Callbacks (The modern JavaScript alternative to Callbacks)

Trong JavaScript hiện đại, chúng ta có các promise objects. Promises có thể là một giải pháp thay thế cho các callbacks cho các API không đồng bộ. Thay vì chuyển một callback như một đối số và xử lý lỗi ở cùng một vị trí, một promise object cho phép chúng ta xử lý các trường hợp thành công và trường hợp lỗi riêng biệt và nó cũng cho phép chúng ta xâu chuỗi (chain) nhiều cuộc gọi không đồng bộ thay vì lồng chúng.

Nếu hàm `readFileAsArray` hỗ trợ các promises, chúng ta có thể sử dụng nó như sau:

```js
readFileAsArray('./numbers.txt')
  .then(lines => {
    const numbers = lines.map(Number);
    const oddNumbers = numbers.filter(n => n%2 === 1);
    console.log('Odd numbers count:', oddNumbers.length);
  })
  .catch(console.error);
```

Thay vì truyền vào một callback function, chúng ta đã gọi một `.then` function trên giá trị trả về của host function. This `.then` function thường cung cấp cho chúng ta quyền truy cập vào cùng lines array mà chúng ta có trong phiên bản callback và chúng ta có thể xử lý nó như trước. Để xử lý lỗi, chúng tôi thêm một lệnh gọi `.catch` vào kết quả và điều đó cho phép chúng ta truy cập vào một lỗi khi nó xảy ra.

Làm cho host function hỗ trợ một promise interface dễ dàng hơn trong JavaScript hiện đại nhờ vào đối tượng Promise mới. Đây là `readFileAsArray` function được sửa đổi để hỗ trợ một promise interface thêm vào callback interface mà nó đã hỗ trợ:

```js
const readFileAsArray = function(file, cb = () => {}) {
  return new Promise((resolve, reject) => {
    fs.readFile(file, function(err, data) {
      if (err) {
        reject(err);
        return cb(err);
      }

      const lines = data.toString().trim().split('\n');
      resolve(lines);
      cb(null, lines);
    });
});
};
```

Như vậy, chúng ta làm cho hàm trả về một Promise object, nó wraps lệnh gọi async `fs.readFile`. The promise object trưng ra hai đối số, một `resolve` function và một `reject` function.

Bất cứ khi nào chúng ta muốn gọi ra (invoke) the callback với một lỗi chúng ta sử dụng promise `reject` function, và bất cứ khi nào chúng ta muốn gọi ra (invoke) the callback với dữ liệu chúng ta sử dụng promise `resolve` function.

Một điều khác duy nhất chúng ta cần làm trong trường hợp này là cần có một giá trị mặc định cho đối số của callback này trong trường hợp mã đang được sử dụng với promise interface. Chúng ta có thể sử dụng đơn giản là, default empty function trong đối số cho trường hợp đó: `() => {}`.

#### Consuming promises with async/await

Thêm một promise interface giúp mã của bạn hoạt động dễ dàng hơn rất nhiều khi có nhu cầu lặp qua (loop over) một async function. Với callbacks, mọi thứ trở nên lộn xộn.

Promises sẽ cải thiện điều đó một chút, và các function generators sẽ cải thiện điều đó thêm một chút nữa. Điều này nói rằng, một sự thay thế gần đây hơn để làm việc với mã async là sử dụng `async` function, cho phép chúng ta xử lý mã async như thể nó synchronous, làm cho nó dễ đọc hơn rất nhiều.

Đây là cách chúng ta có thể sử dụng hàm `readFileAsArray` với async/await:

```js
async function countOdd () {
  try {
    const lines = await readFileAsArray('./numbers');
    const numbers = lines.map(Number);
    const oddCount = numbers.filter(n => n%2 === 1).length;
    console.log('Odd numbers count:', oddCount);
  } catch(err) {
    console.error(err);
  }
}

countOdd();
```

Trước tiên chúng ta tạo một async function, đây chỉ là một normal function có từ `async` trước nó. Bên trong hàm async, chúng ta gọi hàm `readFileAsArray` như thể nó trả về lines variable và để làm cho nó hoạt động, chúng ta sử dụng từ khóa `await`. Sau đó, chúng ta tiếp tục mã như thể cuộc gọi `readFileAsArray` là đồng bộ.

Để có được những thứ đó chạy, chúng ta thực hiện async function. Điều này rất đơn giản và dễ đọc hơn. Để làm việc với các lỗi, chúng ta cần phải wrap the async call trong một `try`/`catch` statement.

Với tính năng async/await này, chúng ta không phải sử dụng bất kỳ API đặc biệt nào (như .then và .catch). Chúng ta chỉ cần dán nhãn các hàm khác nhau và sử dụng JavaScript thuần cho mã.

Chúng ta có thể sử dụng tính năng async/await với bất kỳ function nào hỗ trợ một promise interface. Tuy nhiên, chúng ta không thể sử dụng nó với các callback-style async functions (ví dụ như setTimeout).

### The EventEmitter Module

The EventEmitter là một mô-đun hỗ trợ giao tiếp giữa các đối tượng trong Node. EventEmitter là cốt lõi của kiến trúc hướng sự kiện không đồng bộ Node. Nhiều Node's built-in modules kế thừa từ EventEmitter.

Khái niệm này rất đơn giản: các emitter objects phát ra các sự kiện được đặt tên khiến các listeners đã đăng ký trước đó được gọi. Vì vậy, về cơ bản một emitter object có hai tính năng chính:

- Phát ra tên các sự kiện (Emitting name events).
- Đăng ký và hủy đăng ký các listener functions.

Để làm việc với EventEmitter, chúng ta chỉ cần tạo một class extends EventEmitter.

```js
class MyEmitter extends EventEmitter {

}
```

Các emitter objects là những gì chúng ta khởi tạo từ các EventEmitter-based classes:

```js
const myEmitter = new MyEmitter();
```

Tại bất kỳ thời điểm nào trong vòng đời của các emitter objects đó, chúng ta có thể sử dụng emit function ra để phát ra (emit) bất kỳ sự kiện được đặt tên nào chúng ta muốn.

```js
myEmitter.emit('something-happened');
```

Phát ra (Emitting) một sự kiện là tín hiệu cho thấy một số điều kiện đã xảy ra. Điều kiện này thường là về một sự thay đổi trạng thái (state change) trong emitting object.

Chúng ta có thể thêm các listener functions bằng phương thức `on`, và các listener functions này sẽ được thực thi mỗi khi emitter object phát ra sự kiện tên liên quan của chúng (emits their associated name event).

#### Events !== Asynchrony

Hãy xem một ví dụ:

```js
const EventEmitter = require('events');

class WithLog extends EventEmitter {
  execute(taskFunc) {
    console.log('Before executing');
    this.emit('begin');
    taskFunc();
    this.emit('end');
    console.log('After executing');
  }
}

const withLog = new WithLog();

withLog.on('begin', () => console.log('About to execute'));
withLog.on('end', () => console.log('Done with execute'));

withLog.execute(() => console.log('*** Executing task ***'));
```

Class `WithLog` là một event emitter. Nó định nghĩa instance function `execute`. `execute` function này nhận một đối số, một task function, và wraps việc thực thi của nó bằng các log statements. Nó bắn các sự kiện trước và sau khi thực thi.

Để xem trình tự những gì sẽ xảy ra ở đây, chúng ta đăng ký listeners trên cả hai sự kiện được đặt tên và cuối cùng thực hiện một sample task để kích hoạt (trigger) mọi thứ.

Đây là đầu ra của điều đó:

```
Before executing
About to execute
*** Executing task ***
Done with execute
After executing
```

Điều tôi muốn bạn chú ý về đầu ra ở trên là tất cả đều diễn ra đồng bộ. Không có gì không đồng bộ trong mã này.

- Chúng ta nhận được dòng "Before executing" đầu tiên.
- The `begin` named event sau đó gây ra dòng "About to execute".
- Dòng thực thi thực tế sau đó xuất ra dòng "*** Executing task ***".
- The `end` named event sau đó gây ra dòng "Done with execute"
- Chúng ta nhận được dòng "After executing" cuối cùng.

Cũng giống như các plain-old callbacks, không cho rằng các sự kiện có nghĩa là mã đồng bộ hoặc không đồng bộ.

Điều này rất quan trọng, bởi vì nếu chúng ta chuyển một `taskFunc` không đồng bộ tới `execute`, các sự kiện được phát ra sẽ không còn chính xác nữa.

Chúng ta có thể mô phỏng trường hợp này bằng một cuộc gọi `setImmediate`:

```js
// ...

withLog.execute(() => {
  setImmediate(() => {
    console.log('*** Executing task ***')
  });
});
```

Bây giờ đầu ra sẽ là:

```
Before executing
About to execute
Done with execute
After executing
*** Executing task ***
```

Cái này sai. Các dòng sau cuộc gọi async, đã gây ra các cuộc gọi "Done with execute" và "After executing", không còn chính xác nữa.

Để phát ra một sự kiện sau khi chức năng không đồng bộ được thực hiện, chúng ta sẽ cần kết hợp các callbacks (or promises) với giao tiếp dựa trên sự kiện này. Ví dụ dưới đây chứng minh điều đó.

Một lợi ích của việc sử dụng các sự kiện thay vì các regular callbacks là chúng ta có thể phản ứng với cùng một tín hiệu nhiều lần bằng cách xác định nhiều listeners. Để thực hiện tương tự với các callbacks, chúng ta phải viết nhiều logic hơn trong single available callback. Events là một cách tuyệt vời để các ứng dụng cho phép nhiều plugin bên ngoài xây dựng chức năng trên lõi của ứng dụng. Bạn có thể nghĩ về chúng như các điểm móc nối (hook points) để cho phép tùy chỉnh câu chuyện xung quanh một thay đổi trạng thái.

#### Asynchronous Events

Hãy chuyển đổi synchronous sample example thành một cái gì đó không đồng bộ và hữu ích hơn một chút.

```js
const fs = require('fs');
const EventEmitter = require('events');

class WithTime extends EventEmitter {
  execute(asyncFunc, ...args) {
    this.emit('begin');
    console.time('execute');
    asyncFunc(...args, (err, data) => {
      if (err) {
        return this.emit('error', err);
      }

      this.emit('data', data);
      console.timeEnd('execute');
      this.emit('end');
    });
  }
}

const withTime = new WithTime();

withTime.on('begin', () => console.log('About to execute'));
withTime.on('end', () => console.log('Done with execute'));

withTime.execute(fs.readFile, __filename);
```

The `WithTime` class thực thi một `asyncFunc` và báo cáo thời gian mà `asyncFunc` sử dụng bằng cách sử dụng các lệnh `console.time` và `console.timeEnd`. Nó phát ra đúng trình tự các sự kiện trước và sau khi thực hiện. Và cũng phát ra các error/data events để làm việc với các tín hiệu thông thường của các cuộc gọi không đồng bộ.

Chúng ta kiểm tra `withTime` emitter bằng cách chuyển cho nó một lệnh gọi `fs.readFile`, đây là một hàm không đồng bộ. Thay vì xử lý dữ liệu tệp bằng một callback, bây giờ chúng ta có thể lắng nghe sự kiện data.

Khi chúng ta thực thi mã này, chúng ta nhận được trình tự sự kiện đúng như mong đợi và chúng ta nhận được thời gian báo cáo cho việc thực thi, điều này rất hữu ích:

```
About to execute
execute: 4.507ms
Done with execute
```

Lưu ý cách chúng ta cần kết hợp một callback với một event emitter để thực hiện điều đó. Nếu các `asynFunc` được hỗ trợ promises, chúng ta có thể sử dụng tính năng async/await để thực hiện tương tự:

```js
class WithTime extends EventEmitter {
  async execute(asyncFunc, ...args) {
    this.emit('begin');
    try {
      console.time('execute');
      const data = await asyncFunc(...args);
      this.emit('data', data);
      console.timeEnd('execute');
      this.emit('end');
    } catch(err) {
      this.emit('error', err);
    }
  }
}
```

Tôi không biết bạn thấy thế nào, nhưng điều này đối với tôi dễ đọc hơn nhiều so với mã dựa trên callback-based hoặc bất kỳ dòng .then/.catch nào. Tính năng async/await mang lại cho chúng ta đến gần nhất có thể với chính ngôn ngữ JavaScript, mà tôi nghĩ là một chiến thắng lớn.

#### Events Arguments and Errors

Trong ví dụ trước, có hai sự kiện được phát ra với các đối số phụ.

The error event được phát ra với một error object.

```js
this.emit('error', err);
```

The data event được phát ra với một data object.

```js
this.emit('data', data);
```

Chúng ta có thể sử dụng tùy ý bao nhiêu đối số mà chúng ta cần sau sự kiện được đặt tên và tất cả các đối số này sẽ có sẵn bên trong các listener functions mà chúng ta đăng ký cho các sự kiện được đặt tên này.

Ví dụ, để làm việc với data event, the listener function mà chúng ta đăng ký sẽ có quyền truy cập vào data argument được truyền cho emitted event và data object đó chính xác là những gì `asyncFunc` phơi bày.

```js
withTime.on('data', (data) => {
  // do something with data
});
```

The `error` event thường là một sự kiện đặc biệt. Trong ví dụ dựa trên callback-based của chúng ta, nếu chúng ta không xử lý the error event với listener, the node process sẽ thực sự thoát.

Để chứng minh điều đó, hãy thực hiện một cuộc gọi khác đến phương thức thực thi với một đối số xấu:

```js
class WithTime extends EventEmitter {
  execute(asyncFunc, ...args) {
    console.time('execute');
    asyncFunc(...args, (err, data) => {
      if (err) {
        return this.emit('error', err); // Not Handled
      }

      console.timeEnd('execute');
    });
  }
}

const withTime = new WithTime();

withTime.execute(fs.readFile, ''); // BAD CALL
withTime.execute(fs.readFile, __filename);
```

Cuộc gọi thực thi đầu tiên ở trên sẽ gây ra lỗi. The node process sẽ gặp sự cố và thoát:

```
events.js:163
      throw er; // Unhandled 'error' event
      ^

Error: ENOENT: no such file or directory, open ''
```

Cuộc gọi thực thi thứ hai sẽ bị ảnh hưởng bởi sự cố này và có khả năng sẽ không được thực hiện.

Nếu chúng ta đăng ký một listener cho sự kiện `error` đặc biệt, hành vi của node process sẽ thay đổi. Ví dụ:

```js
withTime.on('error', (err) => {
  // do something with err, for example log it somewhere
  console.log(err)
});
```

Nếu chúng ta làm như trên, lỗi từ cuộc gọi thực thi đầu tiên sẽ được báo cáo nhưng the node process sẽ không gặp sự cố và thoát. Cuộc gọi thực thi khác sẽ kết thúc bình thường:

```
{ Error: ENOENT: no such file or directory, open '' errno: -2, code: 'ENOENT', syscall: 'open', path: '' }
execute: 4.276ms
```

Lưu ý rằng Node hiện hành xử khác với các promise-based functions và chỉ đưa ra cảnh báo, nhưng điều đó cuối cùng sẽ thay đổi:

```
UnhandledPromiseRejectionWarning: Unhandled promise rejection (rejection id: 1): Error: ENOENT: no such file or directory, open ''

DeprecationWarning: Unhandled promise rejections are deprecated. In the future, promise rejections that are not handled will terminate the Node.js process with a non-zero exit code.
```

Một cách khác để xử lý các trường hợp ngoại lệ từ các lỗi phát ra là đăng ký một listener cho global `uncaughtException` process event. Tuy nhiên, bắt lỗi trên toàn cầu với sự kiện đó là một ý tưởng tồi.

Lời khuyên tiêu chuẩn về `uncaughtException` là tránh sử dụng nó, nhưng nếu bạn phải làm (nói để báo cáo những gì đã xảy ra hoặc làm sạch), bạn vẫn nên để process exit:

```js
process.on('uncaughtException', (err) => {
  // something went unhandled.
  // Do any cleanup and exit anyway!

  console.error(err); // don't do just that.

  // FORCE exit the process too.
  process.exit(1);
});
```

Tuy nhiên, hãy tưởng tượng rằng nhiều error events xảy ra cùng một lúc. Điều này có nghĩa là `uncaughtException` listener ở trên sẽ được kích hoạt nhiều lần, đây có thể là một vấn đề đối với một số cleanup code. Một ví dụ về điều này là khi nhiều cuộc gọi được thực hiện cho một database shutdown action.

Mô đun `EventEmitter` bày ra một phương thức `once`. Phương pháp này báo hiệu để gọi ra the listener một lần, không phải mỗi lần nó xảy ra. Vì vậy, đây là trường hợp sử dụng thực tế để sử dụng với `uncaughtException` vì với uncaught exception đầu tiên, chúng ta sẽ bắt đầu cleanup và chúng ta biết rằng dù sao chúng ta cũng sẽ thoát khỏi quy trình.

#### Order of Listeners

Nếu chúng ta đăng ký nhiều listeners cho cùng một sự kiện, việc gọi những listeners đó sẽ theo thứ tự. The first listener mà chúng ta đăng ký là the first listener  được gọi.

```js
// प्रथम
withTime.on('data', (data) => {
  console.log(`Length: ${data.length}`);
});

// दूसरा
withTime.on('data', (data) => {
  console.log(`Characters: ${data.toString().length}`);
});

withTime.execute(fs.readFile, __filename);
```

Đoạn mã trên sẽ khiến dòng "Length" được ghi lại trước dòng "Characters", bởi vì đó là thứ tự mà chúng ta đã xác định những listeners đó.

Nếu bạn cần xác định một listener mới, nhưng để listener đó được gọi ra trước, bạn có thể sử dụng phương thức `prependListener`:

```js
// प्रथम
withTime.on('data', (data) => {
  console.log(`Length: ${data.length}`);
});

// दूसरा
withTime.prependListener('data', (data) => {
  console.log(`Characters: ${data.toString().length}`);
});

withTime.execute(fs.readFile, __filename);
```

Ở trên sẽ làm cho dòng "Characters" được logged đầu tiên.

Và cuối cùng, nếu bạn cần loại bỏ một listener, bạn có thể sử dụng phương thức `removeListener`.

Đó là tất cả những gì tôi có cho chủ đề này. Cảm ơn vì đã đọc! Cho đến lần sau!

* * * * *

Learning React or Node? Checkout my books:

- [Learn React.js by Building Games](http://amzn.to/2peYJZj)
- [Node.js Beyond the Basics](http://amzn.to/2FYfYru)
