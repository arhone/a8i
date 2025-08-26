### Серверная часть (PHP):
```php
class AbstractController
{
    /**
     * Максимальное количество вложенных запросов.
     */
    const ARC_MAX_REQUESTS = 5;

    /**
     * Конструктор.
     */
    public function __construct(
        protected NormalizerInterface $normalizer,
        protected HttpKernelInterface $httpKernel
    ) {}

     /**
     * Пример вашего общего метода для отправки ответов.
     * return $this->resolve($service->read($request)) 
     */
    public function resolve(
        Request $request,
        ResponseDataTransfer $response,
        int $code = 200,
        array $headers = [],
        bool $deprecated = false
    ): Response {
    
        // ... Ваша дополнительная логика ...
  
        $accept = $request->headers->get(
            key: 'Accept',
            default: 'application/json'
        );
        return new Response(
            content: $this->normalizer->normalize(
                $this->arh($accept, $request, $response), // Обработка arc и добавление ars в ответ
                'json', // или msgpack или другое, в зависимости от accept
                [AbstractObjectNormalizer::SKIP_NULL_VALUES => true] // Можно взять из arc.options.skip_null
            ),
            status: $code,
            headers: $headers
        );
    }

    /**
     * Advanced Request Handler.
     * Обработчик параметра (Advanced Request Context) ?arc=*
     */
    protected function arh(string $accept, Request $request, ResponseDataTransfer $response): ResponseDataTransfer
    {
        // Обрабатываем только запросы с параметром arc и кодами ответа от родительского запроса 2xx
        if (
            empty($arc = $request->query->get('arc', null))
            || ($response->status->code < 200 || $response->status->code >= 300)
        ) {
            return $response;
        }

        // Декодируем контекст запроса и получаем ответы
        $arc = msgpack_unpack($this->base64UrlDecode($arc));
        if (is_array($arc) && !empty($arc['requests'])) {

            // Сообщаем об ошибке если количество запросов больше максимального
            if (($arcRequestsCount = count($arc['requests'])) > self::ARC_MAX_REQUESTS) {
                return new ResponseDataTransfer(
                    status: new StatusDataTransfer(
                        code: StatusDataTransfer::CODE_BAD_REQUEST,
                        message: 'Максимальное количество вложенных запросов: ' . self::ARC_MAX_REQUESTS,
                        triggers: [
                            new StatusTriggerDataTransfer(
                                name: 'arc',
                                value: $arcRequestsCount
                            )
                        ]
                    )
                );
            }

            $ars = []; // Advanced Response Section
            // Обрабатываем каждый запрос и добавляем в раздел Advanced Response Section
            foreach ($arc['requests'] as $i => $req) {
                if (
                    !empty($path = $req['path'])
                    && !str_contains($path, 'arc=') // Исключаем рекурсию
                ) {
                    // Подготавливаем запрос
                    $subRequest = Request::create($req['path'], $req['method'] ?? 'GET');
                    // Копируем заголовки исходного запроса
                    $subRequest->headers->replace($request->headers->all());
                    // Выполняем запрос
                    $subResponse = $this->httpKernel->handle($subRequest, HttpKernelInterface::SUB_REQUEST);
                    // Добавляем полученный ответ в раздел Advanced Response Section
                    $ars[$req['name'] ?? $i] = $accept === 'application/msgpack'
                        ? msgpack_unpack($subResponse->getContent())
                        : json_decode($subResponse->getContent(), true)
                    ;
                }
            }

            $response->ars = $ars;
        }

        return $response;
    }
    
    /**
     * Преобразует из base64url в строку.
     */
    public static function base64UrlDecode(string $base64Url): string
    {
        $base64 = strtr($base64Url, '-_', '+/');
        if ($padding = strlen($base64) % 4) {
            $base64 .= str_repeat('=', 4 - $padding);
        }

        return base64_decode($base64);
    }
}
```
