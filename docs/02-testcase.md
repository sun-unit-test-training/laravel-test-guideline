# Cơ bản về test case

## Khái niệm

Test Case là một thuật ngữ phổ biến thường dùng trong các bài Test cụ thể. Nó thường là đơn vị nhỏ nhất của Testing.

Một Test Case sẽ bao gồm các thông tin như requirements testing (các inputs, điều kiện thực thi), test steps, verification steps, prerequisites, outputs, test environment ...

## Viết test case

Trước khi tạo bất cứ Test Cases nào, chúng ta nên xác định rõ giá trị đầu vào của **từng function/method** cần được test.

Các Test Cases phải được thiết kế để có thể cover được hết các sự kết hợp của các giá trị inputs cùng các điều kiện, bao phủ hết các nhánh if/else.

Nhìn chung, chúng ta thường chia test case ra làm 3 loại dựa trên dữ liệu inputs cho Unit Test.

-   **Normal**: Inputs thuộc vào dải dữ liệu bình thường (accepted). Một lượng lớn codes có thể được cover bằng cách chỉ cần chạy **normal** test cases.
-   **Boundary**: Inputs bằng hoặc xấp xỉ giá trị maximum hay minimum. Chúng được sử dụng để phát hiện lỗi tại cận, thay vì tìm kiếm lỗi tại những vị trí ở giữa trong dải input.
-   **Abnormal**: Inputs là không hợp lệ hay không được kỳ vọng, dùng để kiểm tra khả năng handle lỗi.

Ví dụ: _Giả sử như chúng ta có một function để kiểm tra địa chỉ email nhập vào từ user. Độ dài tối đa của email là 50 ký tự._

```php
function validate($email) {
    if (filter_var($email, FILTER_VALIDATE_EMAIL) && strlen($email) <= 50) {
        return true;
    }
    return false;
}

```

Chúng ta nên viết các Test Cases như sau:

### Normal cases

```php
public function test_valid_email_format_and_length()
{
    // Email with length 18 (less than: maximum - 1)
    $email = 'sample@framgia.com';
    $this->assertEquals(true, validate($email));
}
```

### Boundary cases

```php
public function test_valid_email_format_and_length_max_minus()
{
    // Email with length 49 (maximum - 1)
    $email = 'samplesamplesamplesamplesamplesamples@framgia.com';
    $this->assertEquals(true, validate($email));
}

public function test_valid_email_format_and_length_max()
{
    // Email with length 50 (equal maximum)
    $email = 'samplesamplesamplesamplesamplesamplesa@framgia.com';
    $this->assertEquals(true, validate($email));
}

public function test_valid_email_format_and_length_max_plus()
{
    // Email with length 51 (maximum + 1)
    $email = 'samplesamplesamplesamplesamplesamplesam@framgia.com';
    $this->assertEquals(false, validate($email));
}
```

### Abnormal cases

```php
public function test_invalid_email_format()
{
    // Invalid email format with normal length (between 0 ~ 50)
    $email = 'framgia.com';
    $this->assertEquals(false, validate($email));
}

public function test_valid_email_format_and_length_exceeded()
{
    // Email with length 54
    $email = 'samplesamplesamplesamplesamplesamplesample@framgia.com';
    $this->assertEquals(false, validate($email));
}
```
