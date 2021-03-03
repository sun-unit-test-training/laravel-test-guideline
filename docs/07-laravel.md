# Laravel

## Conventions

-   Cấu trúc thư mục
    -   Tất cả Unit Tests được đặt trong thư mục `tests/Unit` (xem config testsuite trong `phpunit.xml`)
    -   Tất cả Integration Tests được đặt trong thư mục `tests/Integration`
    -   Nội dung bên trong thư mục `Unit` có cấu trúc giống với cấu trúc bên trong thư mục `app`. Ví dụ như Unit Test cho file `app/Models/User.php` tương ứng là `tests/Unit/Models/UserTest.php`
-   Quy tắc đặt tên
    -   Thường có namespace bắt đầu với `Tests\` (xem phần `autoload-dev` trong composer.json)
    -   Method test phải được bắt đầu bằng `test`, viết dạng `camelCase` hay `snake_case` đều được, không phải quá lo lắng về tên method test quá dài, nhưng nên chọn 1 trong hai cho thống nhất, prefer `snake_case` để cho dễ đọc hơn:
    ```php
    public function test_it_throws_an_exception_when_email_is_too_long()
    {
    }
    ```

TODO
