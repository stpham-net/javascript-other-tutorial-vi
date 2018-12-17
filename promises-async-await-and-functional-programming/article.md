
# Promises, async/await, and functional programming.

Không giống như nhiều thứ của internet, tôi thích viết Javascript. Tôi cũng là một fan hâm mộ của functional programming; từ quan điểm thực tế, và từ quan điểm thẩm mỹ. Trong nghệ thuật code, functional thật đẹp.

Thật không may, giống như hầu hết các nhà phát triển JS, tình yêu của tôi về functional code là mâu thuẫn với bản chất không đồng bộ của ngôn ngữ.

![][2]

Thành thật mà nói, làm thế nào để tìm thấy một hình ảnh phù hợp đại diện cho functional programming? Hình này khá đẹp.

### Đây là thách thức:

1. Web vốn dĩ là **không đồng bộ**. Chờ đợi các network requests, user input hoặc animations - tất cả chúng cần phải xảy ra mà không giữ phần còn lại của mã của chúng ta.   
_(Tất nhiên: Async không phải là thứ duy nhất cho web. Và như bất kỳ lập trình viên nào đã xử lý các hệ thống đa luồng nói với bạn: Không bao giờ đơn giản.)_
2. Functional programming không tự nhiên ánh xạ tới các tác vụ không đồng bộ. **Tại sao?** Một hàm thuần túy nên có hai thuộc tính:   
A) Nó phải là **quyết đoán**: Đối với một đầu vào cụ thể, nó sẽ luôn tạo ra cùng một đầu ra.  
B) Nó không nên có **tác dụng phụ:** nghĩa là, một chức năng sẽ không ảnh hưởng đến bất cứ điều gì bên ngoài chính nó. Hơn nữa, nó không nên dựa vào bất cứ điều gì trong trạng thái global.

Dưới đây là một số ví dụ:
    
    
    // Functional - Deterministic and side-effect free.
    function add(x, y){
        return x + y;
    }
    // Not functional - Not deterministic (still side effect free)
    function addRand(x){
        return x + Math.random();
    }
    // Not functional - Deterministic, but has side-effects
    var k = 6;
    function addToGlobal(x){
        k += x;
        return k;
    }
    // Not functional - Not deterministic and has side-effects
    var k = 6;
    function addRandToGlobal(x){
        k += (x*Math.random());
        return k;
    }
    

#### Làm thế nào điều này xung đột với lập trình không đồng bộ?

Giả sử bạn muốn tính toán một cái gì đó dựa trên yêu cầu không đồng bộ: giá trị của ví ETH của người dùng (mã thông báo tiền điện tử Ethereum) bằng USD. Bạn sẽ lấy tổng số ETH và nhân nó với tỷ giá hối đoái gần đây nhất - mà bạn nhận được từ một API ở đâu đó.
    
    
    var ethWallet = 2.3;
    function getETHinUSD(){
        let rate = api.ethPrice('USD');
        return ethWallet*rate;
    }
    

Có một vài vấn đề ở đây.

**Đầu tiên** và trước hết, sự kiện mặc dù không có tác dụng phụ, `ethWallet` vẫn là một global variable, điều mà chúng tôi muốn tránh. Thay vào đó, hãy truyền nó cho hàm dưới dạng tham số.
    
    
    function getETHinUSD(ethWallet){
        let rate = api.ethPrice('USD');
        return ethWallet*rate;
    }
    

**Thứ hai,** giả sử hàm `api.ethprice()` không đồng bộ, mã này có thể sẽ chỉ gây ra lỗi. Điều này là do `rate` sẽ không có giá trị cho đến khi `api.ethprice()` kết thúc, nhưng không có gì bảo mã của chúng ta chờ `api.ethprice()`, vì vậy hàm sẽ trả về kết quả của `ethWallet*rate`, trước khi `rate` có giá trị.

Có hai cách để khắc phục điều này. **Promises** hoặc **async/await**.

### **Promises**

Tôi sẽ không đi vào chi tiết về promises trong bài này, ở đây có [thông tin tốt hơn nhiều][3]. Nhưng để nhắc nhở bạn:

> Một promise chỉ đơn giản là một object thể hiện sự hoàn thành cuối cùng (hoặc thất bại) của một hoạt động không đồng bộ và resulting value của nó.

Tóm lại, Promises sẽ hoàn thành (resolve) hoặc fail (reject). Chúng ta có thể làm một cái gì đó khi một promise hoàn thành bằng cách sử dụng hàm `then()` và làm một cái gì đó khi nó thất bại bằng cách sử dụng hàm `catch()`.

Giả sử `api.ethprice()` trả về một Promise. Trong ví dụ cuối cùng, `rate` sẽ trở thành một đối tượng Promise, có hàm `then()`.   
Chúng ta có thể gọi `rate.then( fn(result) )` với callback function gọi là fn sẽ nhận được `result` khi promise hoàn thành.   
**HOẶC LÀ**  
Chúng tôi hoàn toàn có thể coi `api.ethprice()` như một promise và sử dụng trực tiếp `then`:
    
    
    function getETHinUSD(ethWallet){
        return api.ethPrice('USD').then(function(rate){
            return ethWallet*rate;
        })
    }
    

Promises cũng làm điều thú vị được gọi là chaining. Hàm `then` cũng trả về một promise, điều đó có nghĩa là chúng ta có thể thực hiện một việc gì đó một cách không đồng bộ bên trong hàm đó và cũng có một `then`. Chain này cũng cho phép chúng ta xử lý lỗi độc đáo. Chúng ta chỉ phải đặt một `catch` ở cuối và nó sẽ xử lý mọi lỗi không đồng bộ:
    
    
    function getETHinUSD(ethWallet){
        return api.ethPrice('USD')
        .then(function(rate){
            return ethWallet*rate;
        })
        .catch(function(error){
            console.log("Something went wrong:", error);
            return false;
        })
    }
    

Đáng yêu, nhưng vẫn còn một vấn đề: **Nó thậm chí không gần với functional.**

**Quyết đoán?** `getETHinUSD` sẽ trả về một giá trị khác nhau vào các thời điểm khác nhau - ngay cả với cùng một đầu vào. Điều đó ổn nếu chúng ta kỳ vọng tỷ giá USD sẽ thay đổi theo thời gian, nhưng đó không phải là tất cả. Nếu yêu cầu api không thành công, chúng ta sẽ không nhận được cùng một giá trị, nhưng chúng ta thậm chí không nhận được cùng loại dữ liệu!

**Tác dụng phụ?** Hàm `getETHinUSD` tự nó không ảnh hưởng đến trạng thái toàn cầu của chúng ta, nhưng còn chức năng bên trong được truyền cho `then` thì sao? Nó dựa vào giá trị của `ethWallet`, không thể truyền trực tiếp.   
Và những điều gì về api gọi chính nó? Làm thế nào để chúng ta biết chức năng này không ảnh hưởng đến trạng thái toàn cầu của ứng dụng thông qua một số phương pháp khác? Có thể bằng cách yêu cầu giá ETH, nó cập nhật một request counter cho user state của chúng ta, giới hạn số lượng requests chúng ta có thể thực hiện, ngăn chặn truy cập api sau X requests hoặc chỉ làm mất hiệu lực thông tin đăng nhập của chúng ta và đăng xuất chúng ta khỏi toàn bộ api?

### Async/await

Có một cách tiếp cận khác liên quan đến việc tạo một **asynchronous function.** Một lần nữa, các chi tiết nằm ngoài phạm vi của bài viết này, nhưng [được mô tả xuất sắc ở nơi khác][4]. Một lần nữa, để thức tỉnh bộ nhớ của bạn:

> Hàm không đồng bộ là một hàm hoạt động không đồng bộ bằng cách sử dụng cú pháp async/await...   
Sử dụng async functions giống như sử dụng các functions đồng bộ tiêu chuẩn.

Kurzgesagt, chúng ta có thể sử dụng từ khóa `async` để xác định một function sẽ cần chờ một cái gì đó trong body của nó hoàn thành. Trong hàm `async`, chúng tôi sử dụng từ khóa` await` để xác định những gì chúng tôi muốn chờ đợi.

Một lần nữa, bắt đầu với:
    
    
    function getETHinUSD(ethWallet){
        let rate = api.ethPrice('USD');
        return ethWallet*rate;
    }
    

Chúng tôi xác định `getETHinUSD` là một hàm không đồng bộ và chúng tôi chờ kết quả của `api.ethprice()`:
    
    
    async function getETHinUSD(ethWallet){
        let rate = await api.ethPrice('USD');
        return ethWallet*rate;
    }
    

Điều này đã trông thực sự tốt đẹp - phải không?   
Điều gì xảy ra nếu api thất bại? Vâng, chúng tôi phải sử dụng một try/catch.
    
    
    async function getETHinUSD(ethWallet){
        try{
            let rate = await api.ethPrice('USD');
            return ethWallet*rate;
        } catch (error){
            console.log("Something went wrong:", error)
            return false
        }
    }
    

** Đây có phải là quyết đoán?** Không nếu chúng ta giả sử api trả về một giá trị khác theo thời gian và không bao giờ thất bại.

** Có tác dụng phụ?** Có thể, nhưng ít nhất là về mặt trực quan, chúng ta không thấy cùng một inner function truy cập vào phạm vi của parent function. Tuy nhiên, vì async/await vẫn sử dụng các promise dưới mui xe, về mặt kỹ thuật nó vẫn đang diễn ra - nó chỉ không ảnh hưởng đến chúng ta.

Api vẫn có thể có tác dụng phụ, nhưng chúng ta không có lựa chọn nào về điều đó.

### So sánh cạnh nhau

Sử dụng Promises:
    
    
    function getETHinUSD(ethWallet){
        return api.ethPrice('USD')
        .then(function(rate){
            return ethWallet*rate;
        })
        .catch(function(error){
            console.log("Something went wrong:", error);
            return false;
        })
    }
    

Sử dụng async/await:
    
    
    async function getETHinUSD(ethWallet){
        try{
            let rate = await api.ethPrice('USD');
            return ethWallet*rate;
        } catch (error){
            console.log("Something went wrong:", error)
            return false
        }
    }
    

#### Một số quan sát

1. Phiên bản Promise dài hơn. Tôi đã thấy nhiều bài đăng trên blog gọi đây là một chiến thắng cho async/await, nhưng thành thật mà nói, điều đó sẽ không tạo ra nhiều khác biệt trong một codebase của hơn 10.000 dòng.
2. Phiên bản async/await vẫn đang sử dụng promises, chúng chỉ bị ẩn trong cú pháp.
3. Phiên bản async/await trông giống như phiên bản đồng bộ của chúng ta, mà không xử lý cuộc gọi api không đồng bộ. Đây có phải là một chiến thắng? Nhiều blog dường như nghĩ như vậy. Cá nhân tôi thì không.   
Nhìn thoáng qua, thật dễ dàng để nói rằng có một cái gì đó kỳ lạ xảy ra với phiên bản Promise. Sự khác biệt là tinh tế, nhưng ngay lập tức, chúng tôi biết rằng một cái gì đó không đồng bộ đang xảy ra - Theo cấu trúc chứ không phải cú pháp, chúng tôi biết rằng `rate` sẽ không có giá trị ngay lập tức.   
Tất nhiên, tranh luận có thể được đưa ra rằng sự bao gồm theo nghĩa đen của từ 'async` làm cho nó đủ rõ ràng.
4. Trong một chuỗi dài hơn các hoạt động không đồng bộ, hai cách tiếp cận này sẽ trông như thế nào?   
Trong day-job codebase hàng ngày hiện tại của tôi, chúng tôi có một vài key controller functions với nhiều bước không đồng bộ.   
**Hãy tưởng tượng việc tạo một widget mới:**  
a) Bạn cần xác thực người dùng (không đồng bộ)  
b) _sau đó_ Bạn cần thêm widget mới vào cơ sở dữ liệu.  
c) _sau đó_ Bạn cần cập nhật 3 gadget tham chiếu đến widget mới.  
d) _sau đó_ Bạn cần thông báo cho 2 người dùng đang sử dụng mỗi gadget (tổng cộng 6) rằng một widget đã được tạo.  
e) _sau đó_ Bạn cần phải log rằng điều này đã thực hiện thành công.   
** Nếu bất kỳ một trong những bước đó không thành công, chúng tôi muốn ghi lại một different message và role back càng nhiều càng tốt.  
** Sử dụng async/await, có lẽ chúng ta sẽ đặt mỗi cuộc gọi `await` trong một khối` try` duy nhất và xử lý việc catch ở cuối. Nhưng điều gì sẽ xảy ra nếu chúng ta muốn các khối catch riêng biệt cho một vài `awaits` chứ không phải các các cái khác? Hãy tưởng tượng điều đó sẽ trông như thế nào!   
Bây giờ hãy xem xét từng bước xảy ra trong một hàm then() và các lỗi riêng biệt có thể được xử lý bằng các lệnh catch() riêng lẻ, tất cả được nối lại (chained) với nhau.

#### What about being Functional?

Thực tế là nếu bạn có một cuộc gọi api không xác định, bạn chỉ đơn giản là không thể viết một hàm thuần túy. Tuy nhiên, trong JavaScript, functional programming luôn luôn là một tình huống _best-effort_. Ngôn ngữ không được thiết kế để thực sự functional.

Bất kể bạn sử dụng phương pháp nào, bạn có thể tách function code của mình khỏi non-functional code. **Bởi functional là khi bạn có thể duy trì một codebase dễ hiểu, có thể bảo trì.**

Khi bạn nghĩ functional javascript, có lẽ bạn cũng nghĩ rằng _function chaining._ Promise cho phép chúng ta xâu chuỗi các đoạn mã không đồng bộ theo cách functional. Chúng ta có thể xử lý mỗi `then` _giống như_ một hàm được trả về bởi `then` trước đó (ngay cả khi đó thực sự là một phương thức của một đối tượng Promise).
    
    
    <Array>.map().filter().sort()
    <Promise>.then().then().catch().finally()
    

Thấy sự giống nhau không? Sử dụng await khuyến khích bạn không xâu chuỗi các yêu cầu không đồng bộ, thay vào đó sử dụng các giá trị trung gian.
    
    
    let intermediateValue1 = await asynchronousFunction1();
    let intermediateValue2 = await asynchronousFunction2();
    let intermediateValue3 = await asynchronousFunction3();
    

Đó là thủ tục.   
(Không phải là một điều xấu! Nhưng chủ đề của bài này là functional)

Điều này rất hữu ích trong một số trường hợp nhất định, chẳng hạn như sử dụng `intermediateValue1` làm đối số của `asynchronousFunction3()`. Cuối cùng, bạn vẫn lưu trữ kết quả trong một biến trung gian bên ngoài chuỗi promise. Đây là một lợi ích to lớn của async/await - nhưng theo tôi thì nó ít functional hơn.

Tất nhiên, vì dù sao `await` cũng trả lại một promise ngầm, nên bạn có thể biến nó thành chuỗi promise:
    
    
    await asynchronousFunction().then(...);
    

Tốt nhất của cả hai thế giới, hoặc unholy hackery? Tôi sẽ để bạn tự quyết định.

### Phần kết luận

Có vẻ như tôi có một sở thích dành cho Promises, nhưng thành thật mà nói, tất cả chỉ tập trung vào một điều: **Sử dụng công cụ phù hợp cho công việc!**

* Nếu code base của bạn đã sử dụng riêng một cách tiếp cận, hãy tiếp tục sử dụng phương pháp đó. Nó sẽ cứu ai đó khỏi rất nhiều nhầm lẫn.
* Nếu một cách tiếp cận rõ ràng dễ hiểu hơn trong một bối cảnh nhất định, hãy sử dụng giải pháp rõ ràng hơn!
* Nếu nghi ngờ, hãy hỏi một đồng nghiệp. Nếu bạn không có đồng nghiệp (bạn may mắn) hãy thử cả hai và xem cảm giác nào tốt hơn.

Cảm ơn vì đã đọc!

Tôi là [Aidan Breen][5] và tôi điều hành [tư vấn phần mềm][6] tại Dublin, Ireland.

Nếu bạn thích bài đăng này, hãy xem xét đăng ký [danh sách gửi thư cá nhân][7] của tôi để cập nhật ít hơn hàng tháng.

Rất cám ơn [Mike][8] [van][9] [Rossum][10] vì đã chứng minh đọc bài đăng này và cung cấp phản hồi thực sự có giá trị và sâu sắc.

[1]: https://cdn-images-1.medium.com/fit/c/100/100/1*2LYVFYOOK_9qB-eFwMMo5A.jpeg
[2]: https://cdn-images-1.medium.com/max/1600/1*TYLUnXgd_Cpzzka3oxOirQ.jpeg
[3]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
[4]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function
[5]: https://twitter.com/aidanbreen
[6]: http://apbsoftware.ie
[7]: http://Ok,%20screw%20it.%20Let%27s%20give%20this%20a%20go%20and%20see%20what%20happens:%20%20http://eepurl.com/dMEsHg
[8]: https://twitter.com/mikevanrossum
[9]: https://mikevanrossum.nl/
[10]: https://github.com/askmike
