# Checklist

## :white_check_mark: [1] Self-describing test method

> Unit Test method names must be self-describing
>
> Also focus on naming style, keep the naming style consistent across all the test methods and tests.

**Mục đích:**

-   Test case là tài liệu
-   Đọc vào tên method test có thể biết mục đích của test case

**Thực hiện**

-   Tên test method không cần phải quá ngắn gọn
-   Tên test method phải chỉ ra điều kiện và expect của test case
-   Thống nhất convention trong project, mặc định visibility của 1 method trong class là `public` nên có thể loại bỏ từ `public` trong method test

Chọn một trong các convention sau:

1. **[Recommend]** Sử dụng prefix `test_`

    ```php
    function test_it_returns_false_when_input_number_is_odd()
    ```

2. Sử dụng annotation `@test` thì tên test method không cần phải bắt đầu bằng `test_`

    ```php
    /* @test */
    function it_returns_false_when_input_number_is_odd()
    ```

3. Sử dụng `camelCase` thay cho `snake_case`, chỉ nên sử dụng nếu trong project đã viết theo cách này trước đó

    ```php
    function testItReturnsFalseWhenInputNumberIsOdd()
    ```

## :white_check_mark: [2] A3 (Arrange, Asset, Act)

> A3 (Arrange, Asset, Act)
>
> -   Arrange: thiết lập trạng thái, khởi tạo object, giả lập mock
> -   Act: Chạy unit đang cần test (method under test)
> -   Assert: So sánh expected với kết quả trả về

**Mục đích:**

-   Nội dung test method rõ ràng dễ đọc, dễ viết

**Thực hiện**

-   Chia nội dung test làm 3 phần

    ```php
    function test_validation_failed_when_value_exceed_max_length()
    {
        // Arrange
        $username = str_pad('a', UsernameValidation::MAX_LENGTH + 1);

        // Act
        $validation = new UsernameValidation;
        $isValidUsername = $validation->isValid($username);

        // Assert
        $this->assertFalse($isValidUsername);
    }
    ```

-   Ngoại lệ khi test một method throw exception, do vấn đề kỹ thuật nên phải gọi `expectionException()` trước khi gọi method:

    ```php
    function test_it_throws_exception_when_input_is_not_a_number()
    {
        // Assert that
        $this->expectException(InvalidArgumentException::class);

        // Arrange
        $calculator = new Calculator;
        $input1 = 'i am a string';
        $input2 = 100;

        // Act
        $calculator->add($input1, $input2);
    }
    ```

## :white_check_mark: [3] Use sematic/proper assert method

> Keep assert method descriptive. Use proper assert method to improve the readability of code and the error log.

Sử dụng assertion phù hợp

-   Code dễ đọc hiểu hơn
-   Nếu assert failed thì message sinh ra cũng dễ hiểu hơn
-   Signature của các method assert thường có các tham số theo thứ tự là expected value (giá trị mong muốn), actual value (giá trị thực tế khi chạy unit), message (message thông báo nếu fail) nên cần truyền theo thứ tự để PHPUnit có thể generate ra message báo lỗi chính xác

```php
$this->assertEquals($expected, $actual);
// => Failed asserting that $actual matches expected $expected.
$this->assertTrue($expected == $actual);
// => Failed asserting that false is true??

$this->assertSame($expected, $actual);
$this->assertTrue($expected === $actual);

$this->assertContains($element, $array);
$this->assertTrue(in_array($element, $array);

$this->assertCount($expected, $actual);
$this->assertTrue(count($actual) == $expected);

$this->assertInstanceOf(ExpectedClass::class, $actual);
$this->assertTrue($actual instanceOf ExpectedClass);
```

## :white_check_mark: [4] If you write code, write tests

**Thực hiện**

Mọi PR đều phải chú ý đến test

-   PR thêm feature => viết test cho feature mới
-   PR fix bug => viết test để tránh bug xảy ra 1 lần nữa
-   PR refactor => chạy, update test để đảm bảo không phát sinh ảnh hưởng
-   Nên tích hợp CI để chạy test

**FAQ**:

1.  Q: Thời điểm tốt nhất để viết test?

    A: Thời điểm tốt nhất là khi code còn mới! Thời điểm mà cả code và test đều có thể dễ dàng thay đổi. Tưởng tượng code giống như _đất sét_, khi còn mới thì nó mềm và dễ nặn, nếu để lâu thì nó sẽ cứng và dễ vỡ :smile:

## :white_check_mark: [5] Unit vs Integration?

**Rule đơn giản**

-   Unit test
    -   Test từng function hoặc method của một class
    -   Không thực hiện những việc sau:
        -   Truy vấn cơ sở dữ liệu (làm chậm quá trình test)
        -   Sử dụng network (gửi mail, gọi api bên ngoài,...) (làm chậm, kết quả không ổn định vì phụ thuộc vào mạng)
        -   Sử dụng file system (làm chậm quá trình test)
-   Integration test
    -   Test việc kết hợp giữa các unit (function, method) với nhau => test một nhóm Unit (ví dụ test route)
    -   Có thể truy vấn cơ sở dữ liệu (thiết lập một database test riêng biệt)
    -   Có thể sử dụng file system (test việc import/export file, file permission...)

**Lời khuyên**

Với người mới bắt đầu, bạn có thể bắt đầu đi từ integration test bằng việc test từng route.
Trong quá trình viết integration test cố gắng split ra unit test nhỏ hơn nếu được.

Quá nhiều integration test sẽ khiến thời gian chạy test lâu hơn, việc truy vết lỗi cũng khó khăn hơn do 1 feature chạy qua nhiều lớp, layer code.

## :white_check_mark: [6] My tests are fast!

**Thực hiện**

-   Ngoài việc chú trọng vào việc viết test case đúng, cần chú ý đến thời gian chạy test
-   Hạn chế test database (integration), và nếu có thể thì dùng sqlite in-memory làm database test
-   Test không gọi network hay api service ngoài
-   Khi test làm việc với file, cân nhắc sử dụng [vfsStream](https://github.com/bovigo/vfsStream)
-   Khi chạy phpunit để generate code coverage cân nhắc lựa chọn driver thích hợp, xem thêm [Code Coverage](./04-code-coverage.md)

**Tại sao?**

-   Bạn sẽ phải chạy tests thường xuyên, lặp lại => Nếu tests chạy quá chậm sẽ làm ảnh hưởng đến tiến độ, tinh thần làm việc
-   Dự án áp dụng CI để build, test và deploy => Nếu tests chạy quá chậm sẽ dẫn đến việc tích hợp cho cả team bị chậm

## :white_check_mark: [7] Quality over code coverage number!

**Sự thật về code coverage**

-   Không cần viết test đúng vẫn có thể đạt 100% coverage!
-   Có trường hợp đã đạt 100% coverage rồi nhưng vẫn có khả năng lọt bug vì thiếu test case

=> Tham khảo thêm cách phpunit tính coverage => [link](https://github.com/sun7pro/phpunit-training-coverage/issues/2)

**Thực hiện**

-   Chú trọng vào chất lượng test case, viết sao cho đủ test case? làm sao để test chạy nhanh hơn? làm sao để viết test dễ hơn, refactor code?
-   Áp dụng mutation testing vào dự án nếu có thể, để có chỉ số đánh gía tốt hơn => [Mutation Testing](./06-mutation-testing.md)

## Pull request template

Tham khảo [https://github.com/sun7pro/.github/blob/master/PULL_REQUEST_TEMPLATE.md](https://github.com/sun7pro/.github/blob/master/PULL_REQUEST_TEMPLATE.md) để áp dụng pull request template cho dự án.
