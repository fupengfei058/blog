$_POST无法获取到Content-Type是application/json的http请求的body参数，可以通过file_get_contents("php://input")获取到原始的http请求body。针对Content-Type为multipart/form-data类型的请求，从$_POST可以拿到body数据，但却不能通过php://input获取到原始的body数据流。而对于Content-Type为x-www-form-urlencoded的请求，这2者是都可以获取到的。

### [「文章链接」](https://mp.weixin.qq.com/s/SDYKwGUdk85v58WHzVJE_A)
