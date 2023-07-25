# [Создание API gRPC с помощью Rust](https://konghq.com/blog/engineering/building-grpc-apis-with-rust)

# Что такое gRPC?

gRPC — это высокопроизводительная среда удаленного вызова процедур (RPC).

RPC — это протокол вызова удаленных процедур, который одна программа может использовать, чтобы вызвать метод, функцию или процедуру в другой, доступной в сети. 

# Где используют gRPC?

Применяется в микросервисной архитектуре, из-за лучшей пропускной способности за счет передачи бинарных данных формата Protobuf со строгой типизацией, в отличии от текста/JSON в REST API.
Если приложение использует архитектуру микросервисов, в котором многочисленные микросервисы общаются друг с другом, то выбор следует делать в сторону gRPC.

gRPC не поддерживается в браузере, поэтому для монолитного приложения, к которому должен быть клиентский доступ запросов извне или через браузер, следует использовать обычный REST API. 

Также gRPC актуален в случаях, когда необходим стриминг данных так как работает по протоколу HTTP/2.
Недостаток REST API состоит в том, что транспортный протокол, применяемый для передачи данных – HTTP 1.1. Поэтому организация стриминга потоковых данных затруднена.

Клиент и сервер обязаны хранить один и тот же proto файл, он действует как посреднический контракт для клиента для вызова любых доступных функций с сервера.

# Шаги настройки примера

## [Установите компилятор Protocol Buffers Compiler](https://grpc.io/docs/protoc-installation/)

    Он понадобится для генерации кода нашего сервера и клиента.

    ```
    $ apt install -y protobuf-compiler
    $ protoc --version
    ```
## Protobuf определения службы

    ```
    $ cd project
    $ mkdir proto/
    $ touch store.proto
    ... файл project/proto/store.proto заполнить конткрактами 
    ```
## Компиляция protobuf 

    В результате компиляции появится файл src/store.rs

    ```
    file project/build.rs:

        use std::env;
        use std::path::PathBuf;

        fn main() -> Result<(), Box<dyn std::error::Error>> {
        let proto_file = "./proto/store.proto";
        let out_dir = PathBuf::from(env::var("OUT_DIR").unwrap());

        tonic_build::configure()
                .protoc_arg("--experimental_allow_proto3_optional") // for older systems
                .build_client(true)
                .build_server(true)
                .file_descriptor_set_path(out_dir.join("store_descriptor.bin"))
                .out_dir("./src")
                .compile(&[proto_file], &["proto"])?;

        Ok(())
        }
        
    ```

    Мы указали, что хотим построить и клиент, и сервер, и выходным каталогом для сгенерированного кода должен быть каталог src/

    Build:
    ```
    $ cargo build

```

## Тестирование

    ```
    $ cargo build --release --bin cli
    $ cp target/release/cli ./

    $ ./cli add --sku TESTSKU --price 1.99 --quantity 20 --name bananas --description "yellow fruit"
        success: item was added to the inventory.

    $ ./cli get --sku TESTSKU
        found item: { sku: "TESTSKU" }, stock: { price: 1.99, quantity: 0 }, information: { name: "bananas", description: "yellow fruit" }

    $ ./cli add --sku TESTSKU --price 2.99
        Error: Status { code: AlreadyExists, message: "item already exists in inventory" }   

    $ ./cli update-quantity --sku TESTSKU --change -17
        success: quantity was updated. Quantity: 3 Price: 1.99

    $ ./cli update-price --sku TESTSKU --price 2.19
        success: price was updated. Quantity: 3 Price: 2.19   
        


    $ ./cli watch --sku TESTSKU
        streaming changes to item TESTSKU   

        Затем вернитесь в наш предыдущий терминал, внесите несколько изменений и даже полностью удалите элемент:
        $ ./cli update-quantity --sku TESTSKU --change +50
        success: quantity was updated. Quantity: 53 Price: 2.19
        $ ./cli update-price --sku TESTSKU --price 1.99
        success: price was updated. Quantity: 53 Price: 1.99
        $ ./cli remove --sku TESTSKU
        success: item was removed

        $ ./cli watch --sku TESTSKU
        streaming changes to item TESTSKU
        item was updated: Item { identifier: Some(ItemIdentifier { sku: "TESTSKU" }), stock: Some(ItemStock { price: 2.19, quantity: 53 }), information: Some(ItemInformation { name: Some("bananas"), description: Some("yellow fruit") }) }
        item was updated: Item { identifier: Some(ItemIdentifier { sku: "TESTSKU" }), stock: Some(ItemStock { price: 1.99, quantity: 53 }), information: Some(ItemInformation { name: Some("bananas"), description: Some("yellow fruit") }) }
        watched item has been removed from the inventory.
        stream closed


```

## Следующие шаги

[добавление TLS и авторизации](https://grpc.io/docs/guides/auth/) к клиенту и серверу для защиты данных
