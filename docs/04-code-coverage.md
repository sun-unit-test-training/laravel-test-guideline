# Code Coverage

## Khái niệm

**Code coverage** là một phương pháp đánh giá được dùng để mô tả mức độ mà source code của một chương trình đã được thực thi, khi mà một bộ Test cụ thể chạy.

Nói một cách khác, **Code coverage** là một cách để đảm bảo rằng Tests của bạn thực sự đang test Codes của bạn!

Công thức tính Code coverage:

```
Code Coverage = (Tổng số dòng Code được gọi bởi các bài Tests của bạn) / (Tổng số dòng Code trong thực tế) x 100%
```

Ví dụ: Nếu code coverage của bạn là 90%, điều đó có nghĩa là 90% các dòng codes trong project của bạn đã được gọi ghi chạy Test.

## Tạo report với PHPUnit

### Coverage driver

Để generate coverage report bằng PHPUnit, cần có [coverage driver](https://github.com/sebastianbergmann/php-code-coverage/tree/master/src/Driver).

Có 3 drivers bao gồm (ưu tiên từ trên xuống dưới):

-   [pcov](https://github.com/krakjoe/pcov) cho PHPUnit version >= 8
    ```bash
    php -dextension=pcov.so -dpcov.enabled=1 -dpcov.directory=app ./vendor/bin/phpunit --coverage-text
    ```

    **NOTE**: `pcov.directory=app`, trong đó `app` là thư mục chứa source code

-   `phpdbg`
    ```bash
    phpdbg -qrr ./vendor/bin/phpunit --coverage-text
    ```
-   XDebug
    ```bash
    php -dzend_extension=xdebug.so ./vendor/bin/phpunit --coverage-text
    ```

### Coverage format

Có nhiều loại format cho coverage:

```
Code Coverage Options:
  --coverage-clover <file>    Generate code coverage report in Clover XML format
  --coverage-crap4j <file>    Generate code coverage report in Crap4J XML format
  --coverage-html <dir>       Generate code coverage report in HTML format
  --coverage-php <file>       Export PHP_CodeCoverage object to file
  --coverage-text=<file>      Generate code coverage report in text format [default: standard output]
  --coverage-xml <dir>        Generate code coverage report in PHPUnit XML format
```

Nhưng thông dụng nhất là `--coverage-text` thường dùng trong CI hoặc xem nhanh kết quả và `--coverage-html` để xem chi tiết dưới dạng web dashboard và `--coverage-clover` dùng cho CI.

## 100% code coverage

> 100% code coverage is not our main purpose of test!!! 70 - 80% is ok!
>
> Quotes:
>
> -   Quality over quantity
> -   If it scares you then write test for it
> -   Think how to write enough test cases
> -   Think how to write simple code, simple test
> -   Think how to write fast test
