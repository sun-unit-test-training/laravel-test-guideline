# Controller
## Intro

Giả sử bạn có form request như thế này, bạn sẽ test nó như thế nào?

```php
class ProductCreateRequest extends FormRequest
{
    public function rules()
    {
        return [
            'name' => ['required', 'max:255'],
            'sku' => ['required', Rule::unique(Product::getTableName(), 'sku')],
            'image' => ['nullable', 'mimes:jpg,png'],
            'quantity' => ['required', 'integer', 'min:1'],
            'description' => ['required'],
            'short_description' => ['nullable', 'max:255'],
        ];
    }
}
```

```php
class ProductController extends Controller
{
    public function store(ProductCreateRequest $request)
    {
        $inputs = $request->validated();

        $product = $this->productService->create($inputs);

        return back()->with('product', $product);
    }
}
```

Nhắc lại một chút về Form Request:

-   Nhiệm vụ chính của class là dùng để khai báo validation rules cho form
-   Có thể kèm theo việc check authorization
-   (\*1) Có thể kèm theo ["after" validation hook](https://laravel.com/docs/8.x/validation#adding-after-hooks-to-form-requests)
-   (\*2) Có thể kèm theo [`prepareForValidation`](https://laravel.com/docs/8.x/validation#preparing-input-for-validation)
-   Không dùng độc lập mà chỉ inject vào controller (type-hint biến `$request`)
-   Laravel service container khi thực hiện controller action sẽ tự động _resolve_ Form Request dựa vào type hint và tự động thực hiện validate với các rules đã khai báo trong Form Request

Form Request có thể được test thông qua Unit Test hoặc Integration Test (Laravel thường gọi là Feature Test).

## Unit Test

### The first way

Cách tiếp cận đầu tiên đó là test để kiểm tra tất cả rules cần thiết đã được định nghĩa trong Form Request:

```php
function test_it_should_contain_all_the_expected_validation_rules()
{
    $request = new ProductCreateRequest();

    $this->assertEquals([
        'name' => ['required', 'max:255'],
        'sku' => ['required', Rule::unique(Product::getTableName(), 'sku')],
        'image' => ['nullable', 'mimes:jpg,png'],
        'quantity' => ['required', 'integer', 'min:1'],
        'description' => ['required'],
        'short_description' => ['nullable', 'max:255'],
    ], $request->rules());
}
```

Rất đơn giản bạn đã có 100% code coverage cho class `ProductCreateRequest`!

Về lý thuyết thì không sai, function làm gì thì test đúng chức năng của function đấy. Nhưng tôi thì không thích cách này vì một số lý do:

-   Nó giống như đang duplicate code thành 2 nơi
-   Tạo thói quen không tốt, người viết chỉ việc copy code là đạt được coverage, mà chẳng cần để ý xem nó có thật sự chạy đúng không

### The second approach

Cách tiếp cận thứ hai đó là dựa vào document của Laravel về [Manually Creating Validators](https://laravel.com/docs/8.x/validation#manually-creating-validators).

Chúng ta sẽ tự khởi tạo một `Validator` instance với các rules được khai báo trong Form Request:

```php
function test_it_fails_when_name_is_missing()
{
    $request = new ProductCreateRequest();

    $validator = Validator::make([
        'sku' => 'a-product-sku',
        'quantity' => 1,
        'description' => 'I am the master key, buy me!',
    ], $request->rules());

    $this->assertFalse($validator->passes());
    $this->assertContains('title', $validator->errors()->keys());
}
```

Nếu viết theo test case trên, chúng ta sẽ cần rất nhiều test case nữa để test cho từng rule, của từng input:

-   Test name required
-   Test name max length
-   Test sku required
-   Test sku unique
-   Test image type
-   Test quantity required
-   Test quantity is number
-   ...

Nhưng có thể tóm gọn lại bằng cách sử dụng `@dataProvider`, ví dụ:

```php
/**
 * @dataProvider provideInvalidData
 */
function test_invalid_data(array $data)
{
    $request = new ProductCreateRequest();

    $validator = Validator::make($data, $request->rules());

    $this->assertFalse($validator->passes());
}

function makeInvalidData($invalidInputs)
{
    $validInputs = [
        'sku' => 'a-product-sku',
        'quantity' => 1,
        'description' => 'I am the master key, buy me!',
    ];

    return array_merge($validInputs, $invalidInputs);
}

function provideInvalidData()
{
    return [
        [[]], // missing fields
        [$this->makeInvalidData(['name' => ''])],
        [$this->makeInvalidData(['name' => 'name exceed length ' . str_repeat('a', 256)])],
        // How to test file updload?
        [$this->makeInvalidData(['sku' => ''])],
        [$this->makeInvalidData(['sku' => 'existed-sku'])], // How to test Unique rule?
    ];
}
```

Vẫn còn một số câu hỏi bỏ ngõ ở trên :D nhưng chủ yếu là đưa ra ý tưởng test trước.

Nhược điểm lớn nhất của cách này đó là không thể test được hai ý _(\*1)_ và _(\*2)_ nêu ở trên và còn nhiều trường hợp nữa đòi hỏi bạn đào sâu hơn vào cách hoạt động của framework (\*3).

## Integration Test

Trước khi đi vào chi tiết tôi nghĩ cần thống nhất một chút quan điểm về integration test.

Integration test ở đây tức là test tích hợp một nhóm các Unit với nhau. Laravel gọi chung nó là Feature Test, đã có pull request định tách ra làm 3 loại riêng biệt Unit Test, Integration Test và Feature Test ở [đây](https://github.com/laravel/laravel/pull/5169) nhưng cuối cùng vẫn keep như cũ là Unit Test và Feature Test.

Quan điểm ở bài viết này về Integration Test, cụ thể với Laravel đó là:

-   Sử dụng HTTP test để test route (tích hợp controller, form request và framework routing)
-   Có thể sử dụng DB, ví dụ khi test reposity integrate với DB
-   Có thể sử dụng [mock](https://laravel.com/docs/8.x/mocking) để mock queue, storage, mail, repository, service...

Nó vẫn gần gần với Unit Test, chỉ là sử dụng thêm các tính năng, helper của framework để viết test dễ dàng hơn, chưa đến mức test full luồng như một end-user thực sự của hệ thống.

![](https://images.viblo.asia/0f20f8f7-40a1-42b4-a2dc-a181bd998c94.png)

Bắt đầu Integration Test với tài liệu của Laravel: [HTTP Tests](https://laravel.com/docs/8.x/http-tests).

```php
function test_it_fails_when_name_is_missing()
{
    $url = action([ProductController::class, 'store']);

    $response = $this->post($url, [
        'sku' => 'a-product-sku',
        'quantity' => 1,
        'description' => 'I am the master key, buy me!',
    ]);

    $response->assertStatus(422);
    $response->assertSessionHasErrors(['name']);
}
```

Nó khá giống với Unit Test cách 2 ở trên, nhưng có một số điểm khác biệt dễ nhận thấy đó là:

-   Test trông giống thực tế hơn, vì ta thực hiện request trực tiếp đến route url sử dụng helper method `$this->post()` của Laravel. Bạn có thể xem cách Laravel giả lập request ở [đây](https://github.com/laravel/framework/blob/9f986cef11b959679f530eb24d929b39a2690924/src/Illuminate/Foundation/Testing/Concerns/MakesHttpRequests.php#L496), bằng cách sử dụng HTTP Kernel, chứ không phải request đến server qua network!
-   Khắc phục được được vấn đề _(\*3)_ vì luồng xử lý request đã follow theo framework

Tất nhiên, ta có thể áp dụng `@dataProvider` như đã giới thiệu ở trên, nhưng liệu có cách nào tốt hơn không? - Sử dụng multiple data provider.

```php
function makeInvalidData($invalidInputs)
{
    $validInputs = [
        'sku' => 'a-product-sku',
        'quantity' => 1,
        'description' => 'I am the master key, buy me!',
    ];

    return array_filter(array_merge($validInputs, $invalidInputs), function ($value) {
        return $value !== null;
    });
}

/**
 * @dataProvider provideInvalidName
 * @dataProvider provideInvalidQuantity
 * @dataProvider provideInvalidSku
 * @dataProvider provideInvalidImage
 * ...
 */
function test_it_show_error_when_input_invalid($inputKey, $inputValue)
{
    $url = action([ProductController::class, 'store']);
    $inputs = $this->makeInvalidData([
        $inputKey => $inputValue,
    ]);

    $response = $this->post($url, $inputs);

    $response->assertSessionHasErrors([$inputKey]);
}

function provideInvalidName()
{
    return [
        // Tên dataset => dataset value [$inputKey, $inputValue]
        'Name is required' => ['name', null],
        'Name is limit to 255 chars' => ['name', str_repeat('a', 256)],
    ];
}

function provideInvalidQuantity()
{
    return [
        'Quantity is required' => ['quantity', null],
        'Quantity should be integer' => ['quantity', 1.1],
        'Quantity should be greater than 1' => ['quantity', 0],
    ];
}
```

PHPUnit hỗ trợ nhiều data provider cho một test case, nên ý tưởng ở đây là: mỗi input sẽ có một data provider và trong provider chúng ta sẽ mô tả dữ liệu cho case làm validation failed. Với cách này, chúng ta focus vào việc tạo dữ liệu test. Và với việc đặt tên cho mỗi dataset, khi đọc lại code sẽ rất dễ hiểu, cũng như khi một trường hợp fail, PHPUnit sẽ hiển thị message rõ ràng:

```
1) Tests\Feature\ProductControllerTest::test_it_show_error_when_input_invalid with dataset "Quantity should be integer" ('quantity', 1.1)
```

Nếu bạn follow theo convention: Controller chỉ xử lý điều hướng request, logic được xử lý ở Repository hay Service class. Thì việc test unit thuần cho controller là không cần thiết, thay vào đó chúng ta sẽ thực hiện test integration: routing, controller, form request. Việc assert cũng dễ dàng hơn do Laravel đã hỗ trợ sẵn nhiều method [asserts cho HTTP Test](https://laravel.com/docs/8.x/http-tests#available-assertions).

Ở phần trên, chúng ta mới chỉ ví dụ về các input đơn giản, còn 2 vấn đề chưa được nhắc tới đó là test database rule unique và image upload.

### Database rule
Cách đơn giản nhất theo hướng tiếp cận Integration Test đó là sử dụng database test để tạo dữ liệu mẫu theo test case.

Ví dụ ở `ProductCreateRequest` chúng ta có khai báo rule unique cho `sku`, vậy thì cách đơn giản nhất là chuẩn bị database test với 1 bản khi có `sku = existed-sku`, sau đó trong test case bạn sẽ truyền input cho sku là `existed-sku`.

Thông thường factory sẽ được sử dụng để tạo data test cho model.

Vì factory không sử dụng được bên trong data provider của phpunit, nên chúng ta cần thay đổi chút về method test:
```php
/**
 * ...
 * @dataProvider provideInvalidSku
 * ...
 */
function test_it_show_error_when_input_invalid($inputKey, $inputValue)
{
    $url = action([ProductController::class, 'store']);
    $inputs = $this->makeInvalidData([
        $inputKey => is_callable($inputValue) ? $inputValue() : $inputValue,
    ]);

    $response = $this->post($url, $inputs);

    $response->assertSessionHasErrors([$inputKey]);
}

function provideInvalidSku()
{
    return [
        // Tên dataset => dataset value [$inputKey, $inputValue]
        'SKU is required' => ['sku', null],
        'SKU must be unique' => [
            'sku',
            function () {
                Product::factory()->create(['sku' => 'existed-sku']);

                return 'existed-sku';
            },
        ],
    ];
}
```

### File upload

Laravel hỗ trợ tạo fake upload file, chúng ta có thể áp dụng nó như sau:

```php
function provideInvalidImage()
{
    return [
        // Tên dataset => dataset value [$inputKey, $inputValue]
        'Image must be jpg or png' => [
            'image',
            function () {
                Storage::fake();

                return UploadedFile::fake()->image('product.gif');
            },
        ],
    ];
}
```

Rất đơn giản phải không. Tương tự chúng ta sẽ có thêm các case validation success.

## Kết luận
Như các bạn thấy, có nhiều cách để test form validation trong laravel, vì vậy việc làm theo hướng nào phụ thuộc vào chính bạn hay dự án quyết định, bài viết chỉ nêu ra một số cách để bạn tham khảo.
