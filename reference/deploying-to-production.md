*commit/2bdd3999028fba9f31fee30046b1da0414ff109b*

# 部署Rust和WebAssembly到生产环境

> **⚡ 部署Rust和WebAssembly构建的Web应用程序与部署任何其他的Web应用程序几乎是相同！**

在客户端不是一个用Rust生成的WebAssembly的Web应用程序，
拷贝构建的Web应用程序的文件到生产环境服务器的文件系统并配置你的HTTP服务器使他们可以被访问。

## 确保你的HTTP服务器使用了`application/wasm`的MIME类型

为了最快的页面加载，你将需要使用[`WebAssembly.instantiateStreaming`][instantiateStreaming]
函数，通过网络传输来编译和实例化wasm（或者确保你的绑定能够使用那个函数）。
然而，`instantiateStreaming`要求HTTP响应拥有`application/wasm` [MIME类型][]设置，
否则它将会抛出一个错误。

* [如何为Apache HTTP服务器配置MIME类型][apache-mime]
* [如何为NGINX HTTP服务器配置MIME类型][nginx-mime]

[instantiateStreaming]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/instantiateStreaming
[MIME type]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types
[apache-mime]: https://httpd.apache.org/docs/2.4/mod/mod_mime.html#addtype
[nginx-mime]: https://nginx.org/en/docs/http/ngx_http_core_module.html#types

## 更多资源

* [在生产环境中针对Webpack的最好实践。][webpack-prod]
  在Rust和WebAssembly项目中使用Webpack来绑定他们的Rust生成的WebAssembly，JavaScript，CSS和HTML。
  这个指南有对于在部署到生产环境时充分利用Webpack的提示。
* [Apache 文档。][apache] Apache是一个用于生产环境的HTTP服务器。
* [NGINX 文档。][nginx] NGINX是一个用户生产环境的HTTP服务器。

[webpack-prod]: https://webpack.js.org/guides/production/
[apache]: https://httpd.apache.org/docs/
[nginx]: https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/
