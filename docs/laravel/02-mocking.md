# Mocking
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

## Mockery

Xem thêm về [Mockery](../read-more/mockery.md) tại [đây](../read-more/mockery.md)
