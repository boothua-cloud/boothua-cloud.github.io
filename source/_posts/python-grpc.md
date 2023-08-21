---
title: Python集成Grpc-FasApi+SSL认证+签名认证
tags:
  - Python
  - Grpc
categories:
  - Python
  - Grpc
index_img: /img/pages/grpc-icon-color.jpeg
abbrlink: ef6828ac
date: 2023-08-21 21:04:16
---

### 前言
在FastApi项目种用到了Grpc，主要用于系统和系统之间通信；抛弃传统Restful的方式，使用Grpc进行通信；Grpc需要随FastApi启动，并且还需动态读取ssl证书，以及签名，特此在这里记录一下。
<!-- more -->

#### 统一说明
> 本项目基于FastApi，其他框架集成方式大同小异
> 下面的node_code是各个节点的名称，用作系统之间通信的唯一标识。
> 签名只是做了简单的方法校验，如果需要校验参数之类的，需要自行改动

### 核心代码

#### Grpc服务端

```python
from concurrent.futures import ThreadPoolExecutor

import grpc

from common.core.logger import logger
from common.settings import settings
from common.utils.file_util import read_file
from rpc_router.interceptor.server_interceptor import SignatureValidationInterceptor
from rpc_router.pb2 import (port_pb2_grpc)
from rpc_router.service.rpc_port_service import PortService

CERTS_PATH = settings.CERTS_PATH


class RpcService:
    def __init__(self):
        self._host = settings.RPC_HOST + settings.PRC_PORT
        self.server = grpc.aio.server(migration_thread_pool=ThreadPoolExecutor(max_workers=100),
                                      interceptors=(SignatureValidationInterceptor(),))
        self._register_server()

    def _register_server(self):
        """
        注册rpc服务端
        :return:
        """
        port_pb2_grpc.add_PortServerServicer_to_server(PortService(), self.server)

    def _get_certs(self):
        """
        读取ssl证书
        :return:
        """
        server_certificate_key = read_file(filepath=CERTS_PATH, file_name='local.key')
        server_certificate = read_file(filepath=CERTS_PATH, file_name='local.pem')
        return grpc.ssl_server_certificate_configuration([(server_certificate_key, server_certificate)])

    async def start_serve(self):
        """
        启动rpc服务
        """
        try:
            # 动态读取ssl证书
            server_credentials = grpc.dynamic_ssl_server_credentials(self._get_certs(), self._get_certs)
            self.server.add_secure_port(address=self._host, server_credentials=server_credentials)
            await self.server.stop(0)
            await self.server.start()
            logger.info('rpc service is start... host is ( %s )' % self._host)
            await self.server.wait_for_termination()
        except KeyboardInterrupt:
            logger.warning('rpc service will stop...')
            await self.server.stop(0)
        except Exception as e:
            logger.error(f"Error starting RPC server: {e}")
            await self.server.stop(0)

    async def stop_serve(self):
        await self.server.stop(None)
```

#### Grpc服务端签名认证
```python
import base64
import traceback
from typing import Callable, Awaitable

import grpc

from common.core.logger import logger
from common.encrypt.encrypt import Encrypt
from common.settings import settings
from common.utils.file_util import read_yaml
from schemas.certificate import CertificateQueryFilter
from schemas.node_info import NodeInfoFilter
from service.cert.certificate_service import CertificateService
from service.node_info.node_info_service import NodeInfoService

_SIGNATURE_HEADER_KEY = settings.SIGNATURE_HEADER_KEY
_NODE_HEADER_KEY = settings.NODE_HEADER_KEY

SKIP_METHODS = ['AddNode', 'UploadCert', 'KeypairSync']
nodeInfoService = NodeInfoService()
certificateService = CertificateService()


class SignatureValidationInterceptor(grpc.aio.ServerInterceptor):

    def __init__(self):

        def abort(ignored_request, context: grpc.aio.ServicerContext) -> None:
            context.abort(grpc.StatusCode.UNAUTHENTICATED, '身份验证失败！')

        self._abort_handler = grpc.unary_unary_rpc_method_handler(abort)

    async def intercept_service(
            self, continuation: Callable[[grpc.HandlerCallDetails], Awaitable[grpc.RpcMethodHandler]],
            handler_call_details: grpc.HandlerCallDetails
    ) -> grpc.RpcMethodHandler:
        try:
            method_name = handler_call_details.method.split('/')[-1]
            if method_name in SKIP_METHODS:
                return await continuation(handler_call_details)
            invocation_metadata = handler_call_details.invocation_metadata
            # node_code
            node_code = dict(invocation_metadata).get(_NODE_HEADER_KEY)
            node_code = base64.b64decode(node_code).decode('utf-8')
            node_info = await nodeInfoService.get_one_node_info(NodeInfoFilter(node_code=node_code))
            if node_info is None:
                return self._abort_handler
            data = read_yaml(settings.CERTS_PATH, settings.LOCAL_INFO_NAME)
            private_key = data['private_key']
            cert_info = await certificateService.get_one(CertificateQueryFilter(node_code=node_code))
            sign = dict(invocation_metadata).get(_SIGNATURE_HEADER_KEY)
            encrypt = Encrypt(encrypt_type=settings.CERT_METHOD, public_key=cert_info.public_key,
                              private_key=private_key)
            if not encrypt.verify(method_name[::-1], sign):
                return self._abort_handler
            return await continuation(handler_call_details)
        except Exception as e:
            traceback.print_exc()
            logger.error(f'签名验证失败:{str(e)}')
            return self._abort_handler
```

#### Grpc客户端签名认证
```python
import base64

import grpc

_SIGNATURE_HEADER_KEY = settings.SIGNATURE_HEADER_KEY
_NODE_HEADER_KEY = settings.NODE_HEADER_KEY


class AuthGateway(grpc.AuthMetadataPlugin):

    def __call__(self, context: grpc.AuthMetadataContext,
                 callback: grpc.AuthMetadataPluginCallback) -> None:
        """Implements authentication by passing metadata to a callback.

        Implementations of this method must not block.

        Args:
          context: An AuthMetadataContext providing information on the RPC that
            the plugin is being called to authenticate.
          callback: An AuthMetadataPluginCallback to be invoked either
            synchronously or asynchronously.
        """
        # 这里我用的是sm2国密非对称加密算法来进行请求简单的签名
        data = read_yaml(settings.CERTS_PATH, settings.LOCAL_INFO_NAME)
        node_code = data['node_code']
        public_key = data['public_key']
        private_key = data['private_key']

        encrypt = Encrypt(encrypt_type=settings.CERT_METHOD, public_key=public_key, private_key=private_key)
        signature = encrypt.sign(context.method_name[::-1])
        node_code = base64.b64encode(node_code.encode('utf-8'))
        callback(((_SIGNATURE_HEADER_KEY, signature), (_NODE_HEADER_KEY, node_code)), None)
```

#### Grpc客户端统一请求通道封装
```python
from typing import Optional

import grpc

from common.settings import settings
from common.utils.file_util import read_file
from rpc_router.interceptor.auth_metadata import AuthGateway

CERTS_PATH = settings.CERTS_PATH


def composite_credentials(node_code: Optional[str]):
    # 将为每个 RPC 调用调用凭据对象
    call_credentials = grpc.metadata_call_credentials(AuthGateway(), name='auth gateway')
    channel_credential = grpc.ssl_channel_credentials(read_file(filepath=CERTS_PATH, file_name='{node_code}.pem'))
    # 将通道凭据和呼叫凭据组合在一起
    return grpc.composite_channel_credentials(channel_credential, call_credentials)


def create_aio_client_channel(host: Optional[str], node_code: Optional[str]) -> grpc.aio.Channel:
    """
    构建统一aio grpc channel
    """
    return grpc.aio.secure_channel(host, credentials=composite_credentials(node_code))


def create_client_channel(host: Optional[str], node_code: Optional[str]) -> grpc.Channel:
    """
    构建统一grpc channel
    """
    return grpc.secure_channel(host, credentials=composite_credentials(node_code))
```

#### Grpc服务端示例
```python
class PortService(port_pb2_grpc.PortServerServicer):
    async def GetFreePort(self, request, context):
        """
        :param request:
        :param context:
        :return:
        """
        result = {}
        return port_pb2.GetFreePortResp(**json.loads(result)) if result else port_pb2.GetFreePortResp()

```

#### Grpc客户端请求示例
```python
async def get_player_port(rpc_params: RpcSchema, job_code: str) -> Optional[port_pb2.GetFreePortResp]:
    """
    :param rpc_params:
    :param job_code:
    :return:
    """
    async with create_aio_client_channel(rpc_params.rpc_host, rpc_params.node_code) as channel:
        stub = port_pb2_grpc.PortServerStub(channel)
        result = await stub.GetFreePort(port_pb2.GetFreePortReq(job_code=job_code))
    return result

```

#### 启动Grpc服务
> 监听FastApi启动，随FastApi启动而启动
```python
def init_system(app: FastAPI):
    rpc_service = RpcService()

    @app.on_event('startup')
    async def start_event():
              # 启动grpc服务
        loop = asyncio.get_event_loop()
        asyncio.run_coroutine_threadsafe(rpc_service.start_serve(), loop)
    
    @app.on_event('shutdown')
    async def stop_event():
        logger.warning('The RPC service is about to be discontinued...')
        await rpc_service.start_serve()
```