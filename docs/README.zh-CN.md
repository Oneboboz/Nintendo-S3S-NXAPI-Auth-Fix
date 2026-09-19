# 中文说明

这个项目记录一个已经验证可用的 S3S / nxapi Docker 认证兼容方案。

问题的核心不是 Nintendo Account 登录本身，而是旧版 `space4y/nxapi-s3s:0.7.0` 的启动脚本仍然依赖旧 nxapi 的 `session_token:` 输出格式。

正确做法是：

1. 使用新版 nxapi 完成 `nso auth`
2. 使用 `nso user` 验证认证状态
3. 使用 `util update-s3s-token` 生成 `config.txt`
4. 不运行旧镜像的认证入口
5. 直接在旧镜像中运行 `s3s.py --getseed`

这样可以保留旧版 s3s 程序，同时绕过已经过时的认证入口。

**注意：`config.txt` 包含认证信息，绝对不要上传到 GitHub。**
