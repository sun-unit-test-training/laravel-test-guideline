# Laravel Mock
## Giới thiệu
Khi thực hiện viết unit test cho laravel, thì việc thực hiện `mock`(mô phỏng một phần chức năng code) sẽ cực kì cần thiết.

Vì sao ư. Ví dụ nhé. Khi thực hiện test một `Controller`, controller này có thực hiện phát đi một sự kiện (`event`), lúc này việc mock một `event listenter` rất quan trọng để tránh việc truyền đi sự kiện thật và gây ảnh hưởng đến hệ thống.  Việc thực hiện `mock` trong trường hợp này  cho phép bạn chỉ test `HTTP response` của `controller` mà không cần lo đến việc sẽ phải thực thi các `event listener` ở môi trường test ra sao khi mà chúng nên được test với test case riêng.

Laravel cung cấp nhiều phương thức hữu ích để mock `events`, `jobs` và các `facades` khác. Chúng chủ yếu cung cấp một layer tiện ích trên Mockery thay vì việc sẽ phải thực hiện gọi một loạt Mockery method thông thường.

## Mocking Object
Một `mock object` được inject vào ứng dụng thông qua [service container](https://laravel.com/docs/8.x/container) của Laravel, bạn phải bind `mock instance` của mình vào container. Việc này hướng dẫn container sử dụng mock của object thay vì chính object đó.
```php
use App\Service;
use Mockery;
use Mockery\MockInterface;

public function test_something_can_be_mocked()
{
    $this->instance(
        Service::class,
        Mockery::mock(Service::class, function (MockInterface $mock) {
            $mock->shouldReceive('process')->once();
        })
    );
}
```
Để thuận tiện hơn, bạn có thể sử dụng `mock` method của Laravel base test case. Đoạn code dưới đây sẽ tương đương với đoạn ở bên trên:
```php
use App\Service;
use Mockery\MockInterface;

$mock = $this->mock(Service::class, function (MockInterface $mock) {
    $mock->shouldReceive('process')->once();
});
```

Trong trường hợp chỉ cần mock một vào methods của một object, những phương thức không được mock sẽ được thực thi bình thường khi gọi đến, bạn có thể sử dụng phương thức `partialMock`:
```php
use App\Service;
use Mockery\MockInterface;

$mock = $this->partialMock(Service::class, function (MockInterface $mock) {
    $mock->shouldReceive('process')->once();
});
```
Tương tự, nếu muốn [`spy`](#spies) một object, base test case class của Larave cung cấp `spy` method dựa trên phương thức `Mockery::spy`. Spies cũng tương tự như mock tuy nhiên nó sẽ ghi lại bất kì tương tác nào giữa spy và đoạn code đang được test. Cho phép `assertions` sau khi đoạn code đã được thực thi.
```php
use App\Service;

$spy = $this->spy(Service::class);

// ...

$spy->shouldHaveReceived('process');
```
>**Lưu ý:**
>  Mock: assertions trước khi gọi phương thức cần test (có chứa phương thức đã được mock).
>  Spies:  Có thể assertion sau khi thực hiện gọi phương thức cần test.
>
> [Khác biệt giữa mock và spies.](http://docs.mockery.io/en/latest/reference/creating_test_doubles.html#mocks-vs-spies)
## Mock Facade
Không giống như những static method khác, facade (cả realtime facade) có thể được mock. Điều này giúp ta có thể test facade tương tự như như các depedecy injection khác.

Khi test, bạn sẽ muốn mock việc gọi đến một facade của Laravel ở controller. Ví dụ ta có một controller như sau:
```php
<?php

namespace App\Http\Controllers;

use Illuminate\Support\Facades\Cache;

class UserController extends Controller
{
    /**
     * Retrieve a list of all users of the application.
     *
     * @return \Illuminate\Http\Response
     */
    public function index()
    {
        $value = Cache::get('key');

        //
    }
}
```

Với controller như trên ta có thể mock việc gọi `Cache` facade bằng `shouldReceive`, nó sẽ trả về instance của Mockery mock. Vì các facade được quảng lý bởi service container của Laravel nên nó sẽ có tính testability cao hơn so với static class thông thường. Ví dụ thử mock `get` method của `Cache` facade
```php
<?php

namespace Tests\Feature;

use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Foundation\Testing\WithoutMiddleware;
use Illuminate\Support\Facades\Cache;
use Tests\TestCase;

class UserControllerTest extends TestCase
{
    public function testGetIndex()
    {
        Cache::shouldReceive('get')
                    ->once()
                    ->with('key')
                    ->andReturn('value');

        $response = $this->get('/users');

        // ...
    }
}
```
***Lưu ý:** Bạn không nên mock `Request` facade, thay vào đó truyền input vào [Http test method ](https://laravel.com/docs/8.x/http-tests) như `get` hoặc `post`. Tương tự tay vì mock facade `Config` gọi `Config::set` khi thực hiện viết test*
### Facade Spies

Nếu bạn muốn `"spy"` facade của mình, bạn có thể gọi `spy` method tương ứng ở facade.
```php
use Illuminate\Support\Facades\Cache;

public function test_values_are_be_stored_in_cache()
{
    Cache::spy();

    $response = $this->get('/');

    $response->assertStatus(200);

    Cache::shouldHaveReceived('put')->once()->with('name', 'Taylor', 10);
}
```
# Mockery
## Expectation
***Lưu ý:** Để các `expectation` được thực hiện chúng ta phải gọi đến hàm `Mockery::close()`, tối nhất nó nên được để trong một callback method như `teardown` hoặc `_after`(tùy thuộc vào việc ta kết hợp Mockery với framework nào. Với Laravel là `teardown`). Lệnh này dọn dẹp vùng chứa Mockery được sử dụng bởi hàm test hiện tại và sẽ chạy bất kỳ tác vụ nào cho `expectation`*

Khi đã tạo một mock object nghĩa là chúng ta muốn xác định chính xác cách nó hoạt động (nó được gọi như thế nào). Đây chính là việc định nghĩa một `expectation`
### Phương thức
Để nói với test chúng ta sẽ thực hiện gọi một method với tên chỉ định, sử dụng phương thức `shouldReceive`
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method');
```
Đây sẽ là `expectation` mà dựa vào đó tất cả các `kỳ vọng` ràng buộc khác được thêm vào.

Chúng ta có thể định nghĩa nhiều method
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method_1', 'name_of_method_2');
```
Có thể khai báo các  `expectation` cùng với giá trị mà nó trả về
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive([
    'name_of_method_1' => 'return value 1',
    'name_of_method_2' => 'return value 2',
]);
```
Cách khác để thiết lập phương thức và kỳ vọng của nó:
```php
$mock = \Mockery::mock('MyClass', [
    'name_of_method_1' => 'return value 1',
    'name_of_method_2' => 'return value 2'
]);
```
Chúng ta cũng có thể định nghĩa những method không nên được gọi
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldNotReceive('name_of_method');
```
Phương thức này chính là việc gọi ngắn gọn `shouldReceive()->never()`
### Tham số
Với mọi phương thức khai báo kỳ vọng, chúng ta có thể thêm kỳ vọng về tham số được truyền vào:
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->with($arg1, $arg2, ...);
// or
$mock->shouldReceive('name_of_method')
    ->withArgs([$arg1, $arg2, ...]);
```
Để tăng tính linh hoạt, ta có thể sử dụng các `matcher class`. Ví dụ phương thức `\Mockery::any()`sẽ khớp bất kỳ tham số nào được truyền với `with`. Mockery cho phép thư viện `Hamcrest`, ví dụ hàm `anything()` chính là tương đương `\Mockery::any()`.

Một điều quan trọng cần lưu ý, điều này có nghĩa là tất cả các expectation được đính kèm sẽ chỉ apply cho method khi nó gọi chính xác các tham số.
```php
$mock = \Mockery::mock('MyClass');

$mock->shouldReceive('foo')->with('Hello');

$mock->foo('Goodbye'); // throws a NoMatchingExpectationException
```
Điều này cho phép thiết lập các kỳ vọng khác nhau dựa trên tham số được cung cấp cho các cuộc gọi dự kiến.
### Match tham số với closure
Thay vì cung cấp một trình đối khớp cho từng tham số, ta có thể cung cấp một `closure` cho tất cả các tham số được truyền một lúc:
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->withArgs(closure);
```

`Closure` nhận tất cả các tham số được truyền khi gọi đến phương thức. Bằng cách này `expectation` sẽ chỉ được apply cho method có tham số truyền vào thỏa mãn closure
```php
$mock = \Mockery::mock('MyClass');

$mock->shouldReceive('foo')->withArgs(function ($arg) {
    if ($arg % 2 == 0) {
        return true;
    }
    return false;
});

$mock->foo(4); // matches the expectation
$mock->foo(3); // throws a NoMatchingExpectationException
```
### Match tham số với giá trị định sẵn
Chúng ta có thể cung cấp các tham số được mong đợi match với tham số được truyền vào khi một mock method được gọi
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->withSomeOfArgs(arg1, arg2, arg3, ...);
```
Thứ tự của các tham số không quan trọng, nó chỉ check có bao gồm giá trị mong đợi hay không, kiểu giá trị cũng cần phải được match
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('foo')
    ->withSomeOfArgs(1, 2);

$mock->foo(1, 2, 3);  // matches the expectation
$mock->foo(3, 2, 1);  // matches the expectation (passed order doesn't matter)
$mock->foo('1', '2'); // throws a NoMatchingExpectationException (type should be matched)
$mock->foo(3);        // throws a NoMatchingExpectationException
```
### Any / no
Chúng ta có thể khai báo rằng `expectation`match với bất kỳ tham số nào
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->withAnyArgs();
```
Điều này luôn được set mặc định trừ khi có chỉ địng khác.

Ngoài ra chúng ta có thể khai báo `exptation` match với việc gọi phương thức không có đối số.
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->withNoArgs();
```
### Return value `Expectation`
Với mock object. chúng ta có thể khai báo với `Mockery` kết quả trả về của một method với `andReturn()`
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->andReturn($value);
```
Nó thiết lập giá trụ được trả về từ việc gọi phương thức.

Có thể thiết lập kỳ vọng cho nhiều giá trị trả về:
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->andReturn($value1, $value2, ...)
```
Như vậy lần gọi đầu tiên sẽ trả về `$value1` và lần gọi tiếp theo sẽ trả về `$value2`.

Nếu gọi phương thức nhiều lần hơn số return value mà chúng ta đã khai báo, Mockery sẽ trả về giá trị cuối cùng cho bất kỳ lệnh gọi phương thức tiếp theo nào
```php
$mock = \Mockery::mock('MyClass');

$mock->shouldReceive('foo')->andReturn(1, 2, 3);

$mock->foo(); // int(1)
$mock->foo(); // int(2)
$mock->foo(); // int(3)
$mock->foo(); // int(3)
```
Hoặc sử dụng cú pháp
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->andReturnValues([$value1, $value2, ...])
```
Với cú pháp trên, thứ tự trả về được xác định bởi chỉ số của mảng và cũng tương tự cách đầu, giá trị cuối cùng sẽ được apply cho tất cả các lần gọi hàm sau đó.

Hai cú pháp sau đây sẽ chủ yếu để giao tiếp với người đọc test:
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->andReturnNull();
// or
$mock->shouldReceive('name_of_method')
    ->andReturn([null]);
```
Nó đánh dấu những lần gọi phương thức của mock object trả về null hoặc không gì cả.

Đôi khi chúng ta muốn tính kết quả trả về, dựa vào các tham số được truyền, khi ấy chúng ta cần dùng `andReturnUsing()`. Nó nhận nhiều hơn một `closure`.

```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->andReturnUsing(closure, ...);
```
Closure được sắp xếp theo hàng đợi bằng cách truyền chúng dưới dạng tham số cho hàm `andReturn()`.

Đôi khi phương thức sẽ trả về chính một trong các đối số được truyền vào. Khi đó phương thức `andReturnArg()` sẽ hữu ích, tham số được trả về chúng là index trong list tham số
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->andReturnArg(1);
```
Đoạn trên sẽ trả về đối số thứ 2 (có index là 1) từ danh sách các đối số khi thực hiện gọi hàm.

***Lưu ý:** Không thể mix `andReturnUsing()` hoặc `andReturnArg` với `andReturn()`*

Nếu muốn mock `fluid interface`, phương thức sau sẽ hữu dụng:
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->andReturnSelf();
```
Nó thiết lập giá trị trả về là tên class được mock.
### Throw exception
Chúng ta có thể giả lập phương thức sẽ throw exception:
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->andThrow(new Exception);
```
Thay vì một đối đượng, ta cs thể truyền vào một Exception class, message và / hoặc code.
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->andThrow('exception_name', 'message', 123456789);
```
### Set Public Properties
Được sử dụng với một `expectation` khi một phương thức match được gọi, chúng ta có thể đặt thuộc tính public của một mock object bằng cách sử dụng `andSet()` hoặc `set()`.
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->andSet($property, $value);
// or
$mock->shouldReceive('name_of_method')
    ->set($property, $value);
```
Trong trường hợp muốn gọi phương thức thật của mock class và trả về kết quả của nó, phương thức `passthru()` nói với `expectation` bỏ qua một return queue.

Nó cho phép `expectation` match và đếm số lượng xác thực số lần gọi được áp dụng so với các phương thức thực trong khi vẫn gọi các lớp phương thức thực với tham số được kỳ vọng.
### Kỳ vọng số lần gọi
Bên cạnh việc thiết lập `expectation` cho các tham số truyền vào hàm và kết quả trả về của chúng, chúng ta có thể thiết lập kỳ vọng về số lần gọi đến hàm đó.

Khi thiết lập kỳ vọng số lần gọi cho một phương thức không được gọi đến sẽ throw `\Mockery\Expectation\InvalidCountException`.

***Lưu ý:** Phương thức này bắt buộc phải gọi `\Mockery::close()` ở cuối test, chẳng hạn như có thể gọi ở phương thức `teardown` với PHPUnit. Nếu không Mockery sẽ không xác minh các lệnh gọi đối với mock object (vì thế việc count cũng không thể thực hiện)*

Để khai báo phương thức sẽ được gọi 0 hoặc nhiều lần:
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->zeroOrMoreTimes();
```
Điều này cũng là mặc định với tất cả các method.

Để nói với Mockery một số lượng chính xác số lần gọi hàm, ta sẽ sử dụng như sau:
```
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->times($n);
```
Với $n sẽ là số lần hàm được gọi.

Một vài trường hợp phổ biến sẽ có phương thức gọi trực tiếp.

Định nghĩa method mong đợi được gọi một lần
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->once();
```
Với phương thức được gọi hai lần
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->twice();
```
Phương thức không được gọi
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->never();
```
### Count modifier
Mockery bổ sung một số phương thức để thiết lập kỳ vọng cho số lần gọi method

Nếu muốn khai báo số lần tối thiểu một phương thức sẽ được gọi, sử dụng `atLeast()`
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->atLeast()
    ->times(3);
```
Đoạn code trên có nghĩa là phương thức được gọi ít  nhất 3 lần.

Tương tự, chúng ta cũng có thể khai báo cho Mockery biết số lần nhiều nhất một phương thức có thể được gọi với `atMost()`
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->atMost()
    ->times(3);
```
Ngoài ra, để set phạm vi số lần được gọi:
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('name_of_method')
    ->between($min, $max);
```
Bản chất của `between()` chính là việc sử dụng `atLeast()->times($min)->atMost()->times($max)`

## Argument Validation
### Validate tham số
Đây chính là việc match tham số khi tạo một kỳ vọng cho tham số truyền vào phương thức.

Mockery sẽ hỗ trợ thư viện Hamcrest. Các ví dụ dưới đây tìm hiểu về các hàm match của Mockery và hàm tương đồng phía Hamcrest

***Lưu ý:** Nếu bạn không muốn sử dụng hàm global của Hamcrest thì có thể sử dụng class `\Hamcrest\Matchers`. Ví dụ `identicalTo($arg)` chính là `\Hamcrest\Matchers::identicalTo($arg)`*

Trình match phổ biến nhất chính là hàm `with()`
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('foo')
    ->with(1):
```
Mockery sẽ hiểu nó cần được gọi hàm `foo` với tham số kiểu `integer` giá trị `1`. Trong tường hợp này, Mockery đầy tiên sẽ thử so sánh với phép `===`. Nếu nó fail phép thử này Mockery sẽ cố gắng fallback với phép so sánh `==`.

Khi thực hiện match một object Mockery chỉ sử dụng phép so sánh `===`.
```php
$object = new stdClass();
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive("foo")
    ->with($object);

// Hamcrest equivalent
$mock->shouldReceive("foo")
    ->with(identicalTo($object));
```

instance khác của `stdCalss` sẽ không được coi là match.

***Lưu ý:** `Mockery\Matcher\MustBe` sẽ không được sử dụng nữa*

Còn nếu bạn chỉ muốn so sánh `==` cho object thì sẽ phải dùng phương thức `equalTo` của Hamcrest
```php
$mock->shouldReceive("foo")
    ->with(equalTo(new stdClass));
```
Trong trường hợp chúng ta không quan tâm đến kiểu dữ liệu, giá trị của biến được truyền vào, chỉ cần có bất kì tham số nào đó trược truyền vào, sử dụng `any()`
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive("foo")
    ->with(\Mockery::any());

// Hamcrest equivalent
$mock->shouldReceive("foo")
    ->with(anything())
```
### Validate kiểu dữ liệu
Hàm `type()` sẽ nhận một chuỗi, chuỗ đó sẽ được ghép vào `is_` để tạo thành một phép kiểu tra hợp lệ

Để match bất kỳ PHP resource nào ta sẽ truyền `resource` vào hàm `type()`

```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive("foo")
    ->with(\Mockery::type('resource'));

// Hamcrest equivalents
$mock->shouldReceive("foo")
    ->with(resourceValue());
$mock->shouldReceive("foo")
    ->with(typeOf('resource'));
```
Nó sẽ trả về `true` từ một `is_resoruce` được gọi nếu đối số được truyền vào là một resource của PHP. Ví dụ tiếp để dễ hiểu hơn`\Mockery::type('float')` hoặc `floatValue()` và `typeOf('float')` kiểm tra sử dụng `is_float()`, và `\Mockery::type('callable')` hay `callable()` của Hamcrest sử dụng `is_callable()`.


`type()` cũng chấp nhận tên của một class hay một interface được sử dụng trong `instanceOf`. Hàm tương tự của Hamcrest là `anInstanceOf`.

Tham khảo đầy đủ các hàm check tại [php.net](https://www.php.net/manual/en/ref.var.php) và các [hàm của Hamcrest ](https://github.com/hamcrest/hamcrest-php/blob/master/hamcrest/Hamcrest.php)

Nếu muốn thực hiện match đối số một cách phức tạp hơn, `on()` chính là hàm hỗ trợ điều này. Nó chấp nhận `anonymous function` là một đối số được truyền vào.
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive("foo")
    ->with(\Mockery::on(closure));
```
Nếu closure trả về `true` nghĩa là tham số được giả định khớp với kỳ vọng và ngược lại
```php
$mock = \Mockery::mock('MyClass');

$mock->shouldReceive('foo')
    ->with(\Mockery::on(function ($argument) {
        if ($argument % 2 == 0) {
            return true;
        }
        return false;
    }));

$mock->foo(4); // matches the expectation
$mock->foo(3); // throws a NoMatchingExpectationException
```
Không có phiên bản Hamcrest nào cho `on()`.

Ngoài ra, chúng ta có thể sử dụng phương thức `withArgs()`. Closure sẽ kiểm tra các đối số được truyền vào phương thức được kỳ vọng và đối số là khớp nếu closure trả về true.
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive("foo")
    ->withArgs(closure);
```

`Closue` match cũng hỗ trợ tham số là `optional`
```php
$closure = function ($odd, $even, $sum = null) {
    $result = ($odd % 2 != 0) && ($even % 2 == 0);
    if (!is_null($sum)) {
        return $result && ($odd + $even == $sum);
    }
    return $result;
};

$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('foo')->withArgs($closure);

$mock->foo(1, 2); // It matches the expectation: the optional argument is not needed
$mock->foo(1, 2, 3); // It also matches the expectation: the optional argument pass the validation
$mock->foo(1, 2, 4); // It doesn't match the expectation: the optional doesn't pass the validation
```
Nếu muốn so khớp một đối số  với một biểu thức chính quy, Mockery hỗ trợ phương thức `pattern()`
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('foo')
    ->with(\Mockery::pattern('/^foo/'));

// Hamcrest equivalent
$mock->shouldReceive('foo')
    ->with(matchesPattern('/^foo/'));
```
`ducktype()` là một phương thức để match kiểu của class
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('foo')
    ->with(\Mockery::ducktype('foo', 'bar'));
```
Nó sẽ khớp bất kỳ tham số nào là một đối tượng chứa danh sách các chứa danh sách các method đã được cung cấp. Tương tự với `on()`, không có version Hamcrest nào cho `ducktype()`.
### Capturing Arguments
Nếu chúng ta muốn thực hiện nhiều match cho cùng một đối số, `capture` cung cấp một giải pháp để cùng hàm `on()` phục vụ điều đó.
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive("foo")
    ->with(\Mockery::capture($bar));
```
Nó chỉ địng tất cả những đối số nào được truyền cho `foo` vào biến`$bar`, từ đó ta sẽ bổ sung validation sử dụng assertion.
### Bổ sung đối sánh tham số
`not()` sẽ khớp với bất kỳ đối số nó không bằng hoặc không giống với tham số được truyền vào nó
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('foo')
    ->with(\Mockery::not(2));

// Hamcrest equivalent
$mock->shouldReceive('foo')
    ->with(not(2));
```

`anyOf()`  sẽ match nếu như tham số của `expectation` bằng một trong bất kỳ tham số nào của hàm
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('foo')
    ->with(\Mockery::anyOf(1, 2));

// Hamcrest equivalent
$mock->shouldReceive('foo')
    ->with(anyOf(1,2));
```
`notAnyOf()` sẽ ngược lại với `anyOf()` nó sẽ match nếu như tham số của `expectation` không bằng bất kỳ tham số nào của phương thức:
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('foo')
    ->with(\Mockery::notAnyOf(1, 2));
```
`notAnyOf()` sẽ không có bên Hamcrest

`subset()` sẽ match nếu như tham số là một mảng có chứa mảng đã cho.f
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('foo')
    ->with(\Mockery::subset(array(0 => 'foo')));
```

Việc này sẽ thực hiện cả trên cả tên biến và giá trị, nó tương ứng với key và value của mảng tham số.

`contains()` cũng tương tự như `subset()` nhưng sẽ không quan tâm đến tên của key của mảng.
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('foo')
    ->with(\Mockery::contains(value1, value2));
```
`hasKey()` khớp với đối số là một mảng có chứa key name đã cho
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('foo')
    ->with(\Mockery::hasKey(key));
```
`hasValue()` khớp với đối số là một mảng có chứa value đã cho
```php
$mock = \Mockery::mock('MyClass');
$mock->shouldReceive('foo')
    ->with(\Mockery::hasValue(value));
```
## Spies
Spies là một loại test double, tuy nhiên khác với mock ở chỗ nó ghi lại tất cả tương tác giữa nó với hệ thống test (SUT) và cho phép đưa ra các assertion với những tương tác đó sau khi SUT chạy.

Tạo một spy giúp bạn không cần phải thiết lập việc call tất cả các phương thức như mock. Bạn chỉ cần tạo assertion cho việc call một vài phương thức mà bạn quan tâm đến, bởi lẽ không phải phương thức nào cũng ảnh hưởng cho một test case nhất định.

Trong khi với `mock` thì hầu như phải áp dụng style `Arrange-Act-Assert` vì chúng phải khai báo `expect` những hàm được gọi cùng kết quả trả về trước khi gọi hàm thực thi trong test, cuối cùng mới là assert xem những expect đó đã được đáp ứng.
```php
// arrange
$mock = \Mockery::mock('MyDependency');
$sut = new MyClass($mock);

// expect
$mock->shouldReceive('foo')
    ->once()
    ->with('bar');

// act
$sut->callFoo();

// assert
\Mockery::close();
```
Còn `spies` có thể áp dụng linh hoạt cả style `Arrange-Act-Assert` hoặc `Given-When-Then`. Nó cho phép bỏ qua expect và chuyển assertion đến sau act ở SUT, giúp cho test case dễ đọc hiểu hơn.

```php
// arrange
$spy = \Mockery::spy('MyDependency');
$sut = new MyClass($spy);

// act
$sut->callFoo();

// assert
$spy->shouldHaveReceived()
    ->foo()
    ->with('bar');
```
`spies` ít hạn chế hơn so với `mock`, Nó giúp nêu bật mục đích test và hạn chế lộ cấu trúc của SUT.

Tuy nhiên, hạn chế của `spies` là debug. Khi mock bị gọi ngoài expect, nó lập tức throw exception, nó gần như được coi là một trình debug. Với spies nếu có một hàm call sai, nó sẽ không thể có bối cảnh thời gian lập tức như `mock`, nó chỉ đơn giản khẳng định một hàm được gọi.

Cuối cùng, spy không thể định nghĩa giá trị return của hàm, chỉ có mock mới làm được điều đó.

### Một số hàm thông dụng của spies
Để verify một phương thức được gọi trong spy, sử dụng `shouldHaveReceived()`
```php
$spy->shouldHaveReceived('foo');
```
Để verify một phương thức không được gọi, sử dụng `shouldNotHaveReceived()`
```php
$spy->shouldNotHaveReceived('foo');
```
Để đối sánh tham số được truyền vào một hàm với `spies` ta có thể dùng hai cách: sử dùng hàm `with` hoặc sử dụng một mảng gồm các tham số cần cần đối sánh để truyền vào hàm:
```php
$spy->shouldHaveReceived('foo')
    ->with('bar');

// Or

$spy->shouldHaveReceived('foo', ['bar']);
```
Lưu ý cách 1 không thể dùng cho `shouldNotHaveReceived()`. Nếu muốn sử dụng với một hàm không được gọi bạn cần phải sử dụng cách 2:
```php
$spy->shouldNotHaveReceived('foo', ['bar']);
```
Để xác nhận số lần hàm được gọi
```php
$spy->shouldHaveReceived('foo')
    ->with('bar')
    ->twice();
```
### Thay thế cú pháp `shouldReceive`
Kể từ Mockery 1.0.0, hỗ trợ gọi các phương thức tương tự như các phương thức của PHP mà không cần đối số dạng `String` trong `should` method

Với spies nó mới chỉ được áp dụng cho `shouldHaveReceived()`
```php
$spy->shouldHaveReceived()
    ->foo('bar');

// Expect number of call
$spy->shouldHaveReceived()
    ->foo('bar')
    ->twice();
```
Vì một số hạn chế cú pháp này chưa được áp dụng cho phương thức `shouldNotHaveReceived()`.
