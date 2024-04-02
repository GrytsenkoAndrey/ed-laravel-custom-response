# ed-laravel-custom-response
One API Response class to rule all HTTP responses feat. Laravel Responsable

Laravel has a built-in helper that converts the given array to JSON using the json_encode function and sets the Content-Type header to application/json. Here is an example from the documentation:

return response()->json([
    'id' => 100,
    'name' => 'Alexey Shatrov'
]);
This helper also supports HTTP codes, headers, and json_encode flags, making it really useful in many cases! However, there are situations where this approach becomes less convenient.

Imagine, that you need to return a 201 HTTP code after creating a resource. Not a problem, right? Just use the second parameter

return response()->json([
    'id' => 100,
    'name' => 'Alexey Shatrov'
], 201);
Next, imagine that as your API grows, you find the need to implement another modification to the response — for example, you may require to add a JSON_UNESCAPED_UNICODE flag to the json_encode function.

Again, this is not a problem, this modification is simple but time-consuming. You'd have to manually open every response()->json() call and add it as the third parameter.

Additionally, don’t forget to include a response code and headers array where it’s omitted (for PHP < 8.0) or utilize a named argument (for PHP ≥ 8.0)

//Resourse fetched for PHP < 8.0
return response()->json([
    'id' => 100,
    'name' => 'Alexey Shatrov'
], 200, [], JSON_UNESCAPED_UNICODE);

//Resourse fetched for PHP >= 8.0
return response()->json([
    'id' => 100,
    'name' => 'Alexey Shatrov'
], options: JSON_UNESCAPED_UNICODE);
So, when the project is huge and there is a need to modify a response, you can meet is persistent desire to send everything to Mordor, because you need to modify it in many places.

Additionally, the code may appear cumbersome like the beard of an ancient ent when you have a large amount of data to respond with.

Transform the code into a more suitable version
Let’s bring some magic and create a Response class called TheTheOneResponse that will rule over all responses. It’s a joke, of course, but why not? This class will implement a simple logic based on the passed HTTP code:

If the HTTP code indicates success, the class will return $data if needed.
If the HTTP code indicates a client or server error, it will return $errorMessage if needed.
Any other logic here, if necessary, but I’ll stop at this point.
The class can be initialized with any values using a constructor, and a response with a payload will be immediately fired after creation. Here is the full class code in a minimal approach

namespace App\Http\Responses;

use Illuminate\Contracts\Support\Responsable;

class TheOneResponse implements Responsable
{
    protected int $httpCode;
    protected array $data;
    protected string $errorMessage;

    public function __construct(int $httpCode, array $data = [], string $errorMessage = '')
    {
        $this->httpCode = $httpCode;
        $this->data = $data;
        $this->errorMessage = $errorMessage;
    }
    
    public function toResponse($request): \Illuminate\Http\JsonResponse
    {
        $payload = match (true) {
            $this->httpCode >= 500 => ['error_message' => $this->errorMessage],
            $this->httpCode >= 400 => ['error_message' => $this->errorMessage],
            $this->httpCode >= 200 => ['data' => $this->data],
        };

        return response()->json(
            data: $payload,
            status: $this->httpCode,
            options: JSON_UNESCAPED_UNICODE
        );
    }
}
Now you can transform response()->json() call into

return new TheOneResponse(201, 
    [
        'id' => 100,
        'name' => 'Alexey Shatrov'
    ]
);
This approach looks more elegant, additionally, you can control all response logic in one place.

How does it work without a toResponse call? This is Laravel magic. When you return an instance of a class that implements Responsable from a controller method, Laravel automatically invokes the toResponse() method on that class and sends the resulting Response back to the client.

However it may not be very convenient to use codes every time, and there’s no control over the passed parameters. You could easily be ambushed by orcs (make mistakes), like passing a 201 for the $httpCode along with an $errorMessage

Lets add static methods called “Named Constructors” for common responses, which will accept only needed parameters and initialize our class with them

public static function ok(array $data)
{
    return new static(200, $data)
}

public static function created(array $data)
{
    return new static(201, $data)
}

public static function notFound(string $errorMessage = "Item not found")
{
    return new static(404, errorMessage: $errorMessage)
}

...
Example usage

public function controllerAction()
{
    // Your controller logic...
  
    if (!$user) {
        return TheOneResponse::notFound('User not found');
    }

    return TheOneResponse::ok($item);
}

This code is clear and convenient, and the class TheOneRequest itself is easy to work with.

Here is the complete class code with named constructors. Additionally, simple validation logic for the passed $httpCode has been added.

namespace App\Http\Responses;

use Illuminate\Contracts\Support\Responsable;

class TheOneResponse
{
    protected int $httpCode;
    protected array $data;
    protected string $errorMessage;

    public function __construct(int $httpCode, array $data = [], string $errorMessage = '')
    {

        if (! (($httpCode >= 200 && $httpCode <= 300) || ($httpCode >= 400 && $httpCode <= 600))) {
          throw new \RuntimeException($httpCode . ' is not valid');
        }

        $this->httpCode = $httpCode;
        $this->data = $data;
        $this->errorMessage = $errorMessage;

        $this->send();
    }
    
    public function toResponse($request): \Illuminate\Http\JsonResponse
    {
        $payload = match (true) {
            $this->httpCode >= 500 => ['error_message' => $this->errorMessage],
            $this->httpCode >= 400 => ['error_message' => $this->errorMessage],
            $this->httpCode >= 200 => ['data' => $this->data],
            //... add your logic to this block
        };

        return response()->json(
            data: $payload,
            status: $this->httpCode,
            options: JSON_UNESCAPED_UNICODE
        );
    }

    public static function ok(array $data)
    {
        return new static(200, $data)
    }
    
    public static function created(array $data)
    {
        return new static(201, $data)
    }
    
    public static function notFound(string $errorMessage = "Item not found")
    {
        return new static(404, errorMessage: $errorMessage)
    }

    //add any other static methods here

}
Also, you can make __construct private and use only named constructors

