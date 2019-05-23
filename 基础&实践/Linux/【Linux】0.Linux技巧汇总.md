1. linux下curl如何发送get & post 带参数的请求（其他方式可能导致服务端无法获取参数）
curl http://ip:port/service/name/op -X POST -d 'param1=16&param2=0&param3=[{"value":1,"value":1}]'     能正常发送，服务器端能正常获取参数