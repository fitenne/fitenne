---
title: "使用 QUIC 上手 HTTP3 - rfc9114 小记"
date: 2025-08-25T11:09:28+08:00
enableMath: false
toc: true
draft: false
tags:
- QUIC
- HTTP
categories:
- QUIC
- HTTP
summary: 菜，练。
---


# 使用 QUIC 上手 HTTP3 - rfc9114 小记

## 前言

长久以来，我对 HTTP 的认识很大程度上停留在 HTTP/1.1：当我不使用其他 agent，直接面对 HTTP 协议时，直接使用 `socat - tcp_connect` 建议一个 TCP 连接，或者使用 `openssl s_client` 建立一个 TLS 连接，然后我就可以直接在终端手动输入一个简单的 HTTP 请求。

HTTP2 使用二进制传输数据，上面的做法行不通了。到了 HTTP3，这一版本的协议其相比 HTTP1.1 差异更大。本文将直接从 QUIC 传输层的角度，观察基本的 HTTP3 涉及的细节。

## 从 HTTP1.1 和 HTTP2 到 HTTP3

相比 TCP，QUIC 的一个连接可以携带多个有序的字节流，并且通常可以在应用程序中调整流控，HTTP2 的连接复用、流控制被移到了 QUIC 中实现。

HTTP1.1 中，一个请求会独占一个 TCP 连接，直到当前请求完成后，这个连接才可以被复用，进行下一次请求。在 HTTP3 中，每一个请求使用自己独占的 QUIC 流，一个 QUIC 连接包含多个 QUIC 流，这些流可以同时独立地进行传输。

后文将多处例子将基于一个具体的，对 quic.tech:8443 的 HTTP 请求，如果不加额外说明，都是下面的 HTTP 请求：

```shell
curl --verbose --http3-only https://quic.tech:8443/
```

相比 HTTP1.1 的明文文本传输，在 HTTP3 中 HTTP 消息被编码为二进制，划分帧后，再通过 QUIC 传输。这个请求中，真正携带请求信息的部分会被编码为一个包含若干个 HTTP fields 的 HEADERS 帧。HEADERS 帧和 HTTP1.1 的请求头类似，HEADERS 帧值得一提的有

- HEADERS 帧的有效载荷会被 QPACK 压缩。

- HTTP2、HTTP3 中都没有 status line（即 HTTP1.1 请求的第一行），请求的 method、path 这一类的信息被编码为 HEADERS 帧中的 pseudo header。如果我以明文书写 HEADERS 帧中的有效载荷，会是这样：

  ```shell
  :method: GET
  :scheme: https
  :authority: quic.tech:8443
  :path: /
  user-agent: curl/8.14.1
  accept: */*
  ```

综合上面所述，我们对 quic.tech 的请求流程将包含：建立 QUIC 连接，打开一个 QUIC 流，在 QUIC 中写入 HEADERS 帧。这些操作和 HTTP1.1 基本可以一一对应起来，简单！但是，仅仅做这些我们是拿不到期望的 HTTP 响应的，HTTP3 需要客户端做的还有很多其他工作。

首先 HTTP3 需要参与的双方在建立 QUIC 连接后，都要先建立一个 control stream，在 control stream 中发送一个 SETTINGS 帧，并且保持 control stream 不关闭。

其次，为了解压缩 HTTP 响应中 QPACK 压缩的 HEADERS 帧，我们需要准备接受服务端发起的一个单向流，其中将会包含 QPACK 解码器需要的一些指令。我们的请求非常简单，我们这一端的编码器很可能不会需要向服务端发送解码所需的指令；但是如果我们需要发送指令的话，那么我们需要对称地为其创建一个向服务端的单向流。

前文提到的流在 rfc9000、rfc9240 中都有详细的描述。

## 构建

有了上面的知识，我们的 HTTP3 客户端可以有一个大概的雏形了。本文将使用 [aioquic](https://github.com/aiortc/aioquic) 和[ pylsqpack](https://github.com/aiortc/pylsqpack) 进行演示。

后文的代码中将使用包含但可能不限于下面的 `import`：

```python
import asyncio
import logging
import sys
from collections.abc import Callable
from io import BytesIO
from typing import Literal, cast

import aioquic.quic.events as events
import pylsqpack
from aioquic.asyncio.client import connect
from aioquic.asyncio.protocol import QuicConnectionProtocol
from aioquic.h3.connection import H3_ALPN
from aioquic.quic.configuration import QuicConfiguration
from aioquic.quic.packet import QuicErrorCode
```

HTTP3 中很多整数会使用一种变长整数编码，这种编码在 rfc9000 的 section-16 中，我们的代码中会使用下面的简单实现：

```python
class VariableLengthIntegerCodec:
    @staticmethod
    def encode(x: int) -> bytes:
        if x < 2**6:
            return x.to_bytes(1, "big")
        elif x < 2**14:
            v = bytearray(x.to_bytes(2, "big"))
            v[0] |= 0b01000000
            return v
        elif x < 2**30:
            v = bytearray(x.to_bytes(4, "big"))
            v[0] |= 0b10000000
            return v
        elif x < 2**62:
            v = bytearray(x.to_bytes(8, "big"))
            v[0] |= 0b11000000

        assert False, "length too large to encode"

    @staticmethod
    def decode_from(buf: BytesIO) -> tuple[int | None, int]:
        b = buf.read(1)
        if not b:
            return None, 1

        log2len = (b[0] >> 6) & 0b11
        length = 1 << log2len
        assert length in [1, 2, 4, 8], "invalid length"

        c = buf.read(length - 1)
        if len(c) != length - 1:
            return None, 1 + len(c)

        b = (b[0] & 0b00111111).to_bytes(1, "big") + c
        return int.from_bytes(b, "big"), length
```

雏形：

```python
class H3Protocol(QuicConnectionProtocol):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._h3streams_reader: dict[int, FrameReader] = {}

        self._control_stream_id: int | None = None

        self._local_qpack_encoder_stream_id: int | None = None
        self._local_qpack_decoder_stream_id: int | None = None
        self._peer_qpack_encoder_stream_id: int | None = None
        self._peer_qpack_decoder_stream_id: int | None = None
        self._qpack_encoder = pylsqpack.Encoder()
        self._qpack_decoder = pylsqpack.Decoder(0, 0)

        self._h3connected = False

    def _ensure_connected(self):
        if self._h3connected:
            return
        self._connect()

    def _connect(self):
        pass

	def _send_encoder_data(self, data: bytes):
		pass

	def _send_decoder_data(self, data: bytes):
		pass

    def send_request(self):
        """
        发送一个请求
        """
        self._ensure_connected()

        stream_id = self._quic.get_next_available_stream_id(is_unidirectional=False)
        frame = HeadersFrame(stream_id, self._qpack_encoder)
        enc_data = frame.set_headers(
            [
                (k.encode(), v.encode())
                for k, v in [
                    (":method", "GET"),
                    (":scheme", "https"),
                    (":authority", "quic.tech"),
                    (":path", "/"),
                    ("host", "quic.tech:8443"),
                    ("accept", "*/*"),
                ]
            ]
        )
        self._quic.send_stream_data(stream_id, frame.get_bytes(), True)
        self._send_encoder_data(enc_data)
		self._h3streams_reader[stream_id] = FrameReader()

    def quic_event_received(self, event: events.QuicEvent) -> None:
        """
        QUIC 层会通过这个回调通知我们收到的 QUIC 事件
        """
        pass
```

​`HeadersFrame`、`_send_encoder_data`、`_send_decoder_data` 简单封装了与 `pylsqpack` 的交互：

```python
class H3Protocol(QuicConnectionProtocol):
    def _send_encoder_data(self, data: bytes):
        if self._local_qpack_encoder_stream_id is None:
            self._local_qpack_encoder_stream_id = (
                self._quic.get_next_available_stream_id(is_unidirectional=True)
            )
        self._quic.send_stream_data(self._local_qpack_encoder_stream_id, data)

    def _send_decoder_data(self, data: bytes):
        if self._local_qpack_decoder_stream_id is None:
            self._local_qpack_decoder_stream_id = (
                self._quic.get_next_available_stream_id(is_unidirectional=True)
            )
        self._quic.send_stream_data(self._local_qpack_decoder_stream_id, data)

class HeadersFrame:
    def __init__(self, stream_id: int, encoder: pylsqpack.Encoder):
        self._stream_id = stream_id
        self._encoder = encoder
        self._payload = bytearray()

    def set_headers(self, headers: list[tuple[bytes, bytes]]) -> bytes:
        """
        返回需要发送给解码器的数据
        """
        enc_data, b = self._encoder.encode(self._stream_id, headers)
        self._payload = bytearray()
        self._payload += 0x01.to_bytes(1, "big")
        self._payload += VariableLengthIntegerCodec.encode(len(b))
        self._payload += b
        return enc_data

    def get_bytes(self) -> bytes:
        return self._payload
```

我们在 `H3Protocol._connect` 中完成 control stream 的初始化，并创建本地的 qpack 编解码器所需要的流：

```python
class SettingsFrame:
    def __init__(self):
        b = bytearray()
        b += VariableLengthIntegerCodec.encode(0x00)  # stream header
        b += VariableLengthIntegerCodec.encode(0x04)  # frame type
        b += VariableLengthIntegerCodec.encode(0)  # frame length
        self._payload = bytes(b)

    def get_bytes(self) -> bytes:
        return self._payload

def _connect(self):
    stream_id = self._quic.get_next_available_stream_id(is_unidirectional=True)
    self._control_stream_id = stream_id
    self._quic.send_stream_data(stream_id, SettingsFrame().get_bytes())
    logging.info(f"created control stream id: {self._control_stream_id}")

    stream_id = self._quic.get_next_available_stream_id(is_unidirectional=True)
    self._local_qpack_encoder_stream_id = stream_id
    self._quic.send_stream_data(stream_id, VariableLengthIntegerCodec.encode(0x02))
    logging.info(f"created local qpack encoder stream id: {stream_id}")

    stream_id = self._quic.get_next_available_stream_id(is_unidirectional=True)
    self._local_qpack_decoder_stream_id = stream_id
    self._quic.send_stream_data(stream_id, VariableLengthIntegerCodec.encode(0x03))
    logging.info(f"created local qpack decoder stream id: {stream_id}")
```

之后我们处理服务器返回的数据流：

```python
class FrameReader:
    def __init__(self):
        self._buffer = bytearray()
        self._frames: list[tuple[int, bytes]] = []

    def next_frame(self) -> tuple[int, bytes] | None:
        if self._frames:
            return self._frames.pop(0)
        return None

    def feed_data(self, data: bytes):
        self._buffer += data

        while True:
            reader = BytesIO(self._buffer)
            frame_type, l1 = VariableLengthIntegerCodec.decode_from(reader)
            if frame_type is None:
                return None

            frame_length, l2 = VariableLengthIntegerCodec.decode_from(reader)
            if frame_length is None:
                return None

            if len(self._buffer) < l1 + l2 + frame_length:
                return None

            frame_payload = reader.read(frame_length)
            assert len(frame_payload) == frame_length, "frame length mismatch"
            if (frame_type - 0x21) % 0x1F == 0:
                logging.info("received reserved frame, ignoring")
                self._buffer = self._buffer[l1 + l2 + frame_length :]
                continue

            logging.info(f"received frame type: {frame_type}, length: {frame_length}")
            self._buffer = self._buffer[l1 + l2 + frame_length :]
            self._frames.append((frame_type, frame_payload))

class H3Protocol(QuicConnectionProtocol):
    def _is_server_unidir_stream(self, stream_id: int) -> bool:
        """
        rfc9000
        """
        return stream_id % 4 == 3

    def _handle_peer_qpack_encoder_data(self, data: bytes):
        for id in self._qpack_decoder.feed_encoder(data):
            dec_data, headers = self._qpack_decoder.resume_header(id)
            self._quic.send_stream_data(self._local_qpack_decoder_stream_id, dec_data)
            for k, v in headers:
                logging.info(f"header: {k.decode()}: {v.decode()}")

    def _handle_peer_qpack_decoder_data(self, data: bytes):
        self._qpack_encoder.feed_decoder(data)

    def quic_event_received(self, event: events.QuicEvent) -> None:
        """
        QUIC 层会通过这个回调通知我们收到的 QUIC 事件
        """
        logging.info(f"quic event: {event}")
        if not isinstance(event, events.StreamDataReceived):
            return

        stream_id = event.stream_id
        # 我们已知的 streams
        if stream_id in self._h3streams_reader:
            reader = self._h3streams_reader[stream_id]
            reader.feed_data(event.data)
            while r := reader.next_frame():
                frame_type, frame_payload = r
                match frame_type:
                    case 0x01:  # headers
                        logging.info(f"received headers frame on stream {stream_id}")
                        dec_data, headers = self._qpack_decoder.feed_header(
                            stream_id, frame_payload
                        )
                        self._quic.send_stream_data(
                            self._local_qpack_decoder_stream_id, dec_data
                        )
                        for k, v in headers:
                            logging.info(f"header: {k.decode()}: {v.decode()}")
                    case 0x00:  # data
                        logging.info(
                            f"received data frame on stream {stream_id}: {frame_payload}"
                        )
            return

        if stream_id == self._peer_qpack_decoder_stream_id:
            self._handle_peer_qpack_decoder_data(event.data)
            return

        if stream_id == self._peer_qpack_encoder_stream_id:
            self._handle_peer_qpack_encoder_data(event.data)
            return

        # 我们第一次收到这个 stream 的数据
        assert self._is_server_unidir_stream(stream_id)
        stream_header, length = VariableLengthIntegerCodec.decode_from(
            BytesIO(event.data)
        )
        if stream_header is None:
            raise Exception("failed to decode stream header")
        match stream_header:
            case 0x00:
                logging.info(f"received server control stream: {stream_id}")
                fr = FrameReader()
                fr.feed_data(event.data[length:])
                self._h3streams_reader[stream_id] = fr
            case 0x01:
                logging.info(f"received server push stream: {stream_id}")
                fr = FrameReader()
                fr.feed_data(event.data[length:])
                self._h3streams_reader[stream_id] = fr
            case 0x02:
                logging.info(f"received peer qpack encoder stream: {stream_id}")
                self._peer_qpack_encoder_stream_id = stream_id
                self._handle_peer_qpack_encoder_data(event.data[length:])
            case 0x03:
                logging.info(f"received peer qpack decoder stream: {stream_id}")
                self._peer_qpack_decoder_stream_id = stream_id
                self._handle_peer_qpack_decoder_data(event.data[length:])
            case _:
                if (stream_header - 0x21) % 0x1F == 0:
                    logging.info(
                        f"received reserved unidirectional stream: {stream_id}"
                    )
                    self._quic.stop_stream(stream_id, error_code=QuicErrorCode.NO_ERROR)
                else:
                    raise Exception(f"unknown unidirectional stream: {stream_id}")
```

最后当我们收到响应后，可以关闭 QUIC 连接：

```python
def quic_event_received(self, event: events.QuicEvent) -> None:
......
    while r := reader.next_frame():
        frame_type, frame_payload = r
        match frame_type:
            case 0x01:  # headers
                logging.info(f"received headers frame on stream {stream_id}")
                dec_data, headers = self._qpack_decoder.feed_header(
                    stream_id, frame_payload
                )
                self._quic.send_stream_data(
                    self._local_qpack_decoder_stream_id, dec_data
                )
                for k, v in headers:
                    logging.info(f"header: {k.decode()}: {v.decode()}")
            case 0x00:  # data
                logging.info(
                    f"received data frame on stream {stream_id}: {frame_payload}"
                )
                self._quic.close()
......

async def main():
    configuration = QuicConfiguration(
        is_client=True,
        alpn_protocols=H3_ALPN,
        secrets_log_file=sys.stderr,
    )

    async with connect(
        "quic.tech",
        8443,
        configuration=configuration,
        create_protocol=H3Protocol,
        wait_connected=True,
    ) as transport:
        transport = cast(H3Protocol, transport)

        transport.send_request()
        await transport.wait_closed()


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO, stream=sys.stderr)
    asyncio.run(main())
```

把所有的代码合到一起：

```python
import asyncio
import logging
import sys
from io import BytesIO
from typing import cast

import aioquic.quic.events as events
import pylsqpack
from aioquic.asyncio.client import connect
from aioquic.asyncio.protocol import QuicConnectionProtocol
from aioquic.h3.connection import H3_ALPN
from aioquic.quic.configuration import QuicConfiguration
from aioquic.quic.packet import QuicErrorCode


class VariableLengthIntegerCodec:
    @staticmethod
    def encode(x: int) -> bytes:
        if x < 2**6:
            return x.to_bytes(1, "big")
        elif x < 2**14:
            v = bytearray(x.to_bytes(2, "big"))
            v[0] |= 0b01000000
            return v
        elif x < 2**30:
            v = bytearray(x.to_bytes(4, "big"))
            v[0] |= 0b10000000
            return v
        elif x < 2**62:
            v = bytearray(x.to_bytes(8, "big"))
            v[0] |= 0b11000000

        assert False, "length too large to encode"

    @staticmethod
    def decode_from(buf: BytesIO) -> tuple[int | None, int]:
        b = buf.read(1)
        if not b:
            return None, 1

        log2len = (b[0] >> 6) & 0b11
        length = 1 << log2len
        assert length in [1, 2, 4, 8], "invalid length"

        c = buf.read(length - 1)
        if len(c) != length - 1:
            return None, 1 + len(c)

        b = (b[0] & 0b00111111).to_bytes(1, "big") + c
        return int.from_bytes(b, "big"), length


class SettingsFrame:
    def __init__(self):
        b = bytearray()
        b += VariableLengthIntegerCodec.encode(0x00)  # stream header
        b += VariableLengthIntegerCodec.encode(0x04)  # frame type
        b += VariableLengthIntegerCodec.encode(0)  # frame length
        self._payload = bytes(b)

    def get_bytes(self) -> bytes:
        return self._payload


class HeadersFrame:
    def __init__(self, stream_id: int, encoder: pylsqpack.Encoder):
        self._stream_id = stream_id
        self._encoder = encoder
        self._payload = bytearray()

    def set_headers(self, headers: list[tuple[bytes, bytes]]) -> bytes:
        """
        返回需要发送给解码器的数据
        """
        enc_data, b = self._encoder.encode(self._stream_id, headers)
        self._payload = bytearray()
        self._payload += 0x01.to_bytes(1, "big")
        self._payload += VariableLengthIntegerCodec.encode(len(b))
        self._payload += b
        return enc_data

    def get_bytes(self) -> bytes:
        return self._payload


class FrameReader:
    def __init__(self):
        self._buffer = bytearray()
        self._frames: list[tuple[int, bytes]] = []

    def next_frame(self) -> tuple[int, bytes] | None:
        if self._frames:
            return self._frames.pop(0)
        return None

    def feed_data(self, data: bytes):
        self._buffer += data

        while True:
            reader = BytesIO(self._buffer)
            frame_type, l1 = VariableLengthIntegerCodec.decode_from(reader)
            if frame_type is None:
                return None

            frame_length, l2 = VariableLengthIntegerCodec.decode_from(reader)
            if frame_length is None:
                return None

            if len(self._buffer) < l1 + l2 + frame_length:
                return None

            frame_payload = reader.read(frame_length)
            assert len(frame_payload) == frame_length, "frame length mismatch"
            if (frame_type - 0x21) % 0x1F == 0:
                logging.info("received reserved frame, ignoring")
                self._buffer = self._buffer[l1 + l2 + frame_length :]
                continue

            logging.info(f"received frame type: {frame_type}, length: {frame_length}")
            self._buffer = self._buffer[l1 + l2 + frame_length :]
            self._frames.append((frame_type, frame_payload))


class H3Protocol(QuicConnectionProtocol):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._h3streams_reader: dict[int, FrameReader] = {}

        self._control_stream_id: int | None = None

        self._local_qpack_encoder_stream_id: int | None = None
        self._local_qpack_decoder_stream_id: int | None = None
        self._peer_qpack_encoder_stream_id: int | None = None
        self._peer_qpack_decoder_stream_id: int | None = None
        self._qpack_encoder = pylsqpack.Encoder()
        self._qpack_decoder = pylsqpack.Decoder(0, 0)

        self._h3connected = False

    def _ensure_connected(self):
        if self._h3connected:
            return
        self._connect()

    def _connect(self):
        stream_id = self._quic.get_next_available_stream_id(is_unidirectional=True)
        self._control_stream_id = stream_id
        self._quic.send_stream_data(stream_id, SettingsFrame().get_bytes())
        logging.info(f"created control stream id: {self._control_stream_id}")

        stream_id = self._quic.get_next_available_stream_id(is_unidirectional=True)
        self._local_qpack_encoder_stream_id = stream_id
        self._quic.send_stream_data(stream_id, VariableLengthIntegerCodec.encode(0x02))
        logging.info(f"created local qpack encoder stream id: {stream_id}")

        stream_id = self._quic.get_next_available_stream_id(is_unidirectional=True)
        self._local_qpack_decoder_stream_id = stream_id
        self._quic.send_stream_data(stream_id, VariableLengthIntegerCodec.encode(0x03))
        logging.info(f"created local qpack decoder stream id: {stream_id}")

    def _send_encoder_data(self, data: bytes):
        if self._local_qpack_encoder_stream_id is None:
            self._local_qpack_encoder_stream_id = (
                self._quic.get_next_available_stream_id(is_unidirectional=True)
            )
        self._quic.send_stream_data(self._local_qpack_encoder_stream_id, data)

    def _send_decoder_data(self, data: bytes):
        if self._local_qpack_decoder_stream_id is None:
            self._local_qpack_decoder_stream_id = (
                self._quic.get_next_available_stream_id(is_unidirectional=True)
            )
        self._quic.send_stream_data(self._local_qpack_decoder_stream_id, data)

    def send_request(self):
        """
        发送一个请求
        """
        self._ensure_connected()

        stream_id = self._quic.get_next_available_stream_id(is_unidirectional=False)
        frame = HeadersFrame(stream_id, self._qpack_encoder)
        enc_data = frame.set_headers(
            [
                (k.encode(), v.encode())
                for k, v in [
                    (":method", "GET"),
                    (":scheme", "https"),
                    (":authority", "quic.tech"),
                    (":path", "/"),
                    ("host", "quic.tech:8443"),
                    ("accept", "*/*"),
                ]
            ]
        )
        self._quic.send_stream_data(stream_id, frame.get_bytes(), True)
        self._send_encoder_data(enc_data)
        self._h3streams_reader[stream_id] = FrameReader()

    def _is_server_unidir_stream(self, stream_id: int) -> bool:
        """
        rfc9000
        """
        return stream_id % 4 == 3

    def _handle_peer_qpack_encoder_data(self, data: bytes):
        for id in self._qpack_decoder.feed_encoder(data):
            dec_data, headers = self._qpack_decoder.resume_header(id)
            self._quic.send_stream_data(self._local_qpack_decoder_stream_id, dec_data)
            for k, v in headers:
                logging.info(f"header: {k.decode()}: {v.decode()}")

    def _handle_peer_qpack_decoder_data(self, data: bytes):
        self._qpack_encoder.feed_decoder(data)

    def quic_event_received(self, event: events.QuicEvent) -> None:
        """
        QUIC 层会通过这个回调通知我们收到的 QUIC 事件
        """
        logging.info(f"quic event: {event}")

        if isinstance(event, events.StopSendingReceived):
            self._quic.stop_stream(event.stream_id, error_code=event.error_code)
            return

        if not isinstance(event, events.StreamDataReceived):
            return

        stream_id = event.stream_id
        # 我们已知的 streams
        if stream_id in self._h3streams_reader:
            reader = self._h3streams_reader[stream_id]
            reader.feed_data(event.data)
            while r := reader.next_frame():
                frame_type, frame_payload = r
                match frame_type:
                    case 0x01:  # headers
                        logging.info(f"received headers frame on stream {stream_id}")
                        dec_data, headers = self._qpack_decoder.feed_header(
                            stream_id, frame_payload
                        )
                        self._quic.send_stream_data(
                            self._local_qpack_decoder_stream_id, dec_data
                        )
                        for k, v in headers:
                            logging.info(f"header: {k.decode()}: {v.decode()}")
                    case 0x00:  # data
                        logging.info(
                            f"received data frame on stream {stream_id}: {frame_payload}"
                        )
                        self._quic.close()
            return

        if stream_id == self._peer_qpack_decoder_stream_id:
            self._handle_peer_qpack_decoder_data(event.data)
            return

        if stream_id == self._peer_qpack_encoder_stream_id:
            self._handle_peer_qpack_encoder_data(event.data)
            return

        # 我们第一次收到这个 stream 的数据
        assert self._is_server_unidir_stream(stream_id)
        stream_header, length = VariableLengthIntegerCodec.decode_from(
            BytesIO(event.data)
        )
        if stream_header is None:
            raise Exception("failed to decode stream header")
        match stream_header:
            case 0x00:
                logging.info(f"received server control stream: {stream_id}")
                fr = FrameReader()
                fr.feed_data(event.data[length:])
                self._h3streams_reader[stream_id] = fr
            case 0x01:
                logging.info(f"received server push stream: {stream_id}")
                fr = FrameReader()
                fr.feed_data(event.data[length:])
                self._h3streams_reader[stream_id] = fr
            case 0x02:
                logging.info(f"received peer qpack encoder stream: {stream_id}")
                self._peer_qpack_encoder_stream_id = stream_id
                self._handle_peer_qpack_encoder_data(event.data[length:])
            case 0x03:
                logging.info(f"received peer qpack decoder stream: {stream_id}")
                self._peer_qpack_decoder_stream_id = stream_id
                self._handle_peer_qpack_decoder_data(event.data[length:])
            case _:
                if (stream_header - 0x21) % 0x1F == 0:
                    logging.info(
                        f"received reserved unidirectional stream: {stream_id}"
                    )
                    self._quic.stop_stream(stream_id, error_code=QuicErrorCode.NO_ERROR)
                else:
                    raise Exception(f"unknown unidirectional stream: {stream_id}")


async def main():
    configuration = QuicConfiguration(
        is_client=True,
        alpn_protocols=H3_ALPN,
        secrets_log_file=sys.stderr,
    )

    async with connect(
        "quic.tech",
        8443,
        configuration=configuration,
        create_protocol=H3Protocol,
        wait_connected=True,
    ) as transport:
        transport = cast(H3Protocol, transport)

        transport.send_request()
        await transport.wait_closed()


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO, stream=sys.stderr)
    asyncio.run(main())

```

## HTTP3 要点

建立 QUIC 连接后，HTTP3 两端分别首先需要打开、保持一个 control stream 并发送一个 SETTINGS 帧。

使用 pseudo headers 编码请求方法、路径、schema 等信息。

HEADERS 帧需要进行 QPACK 压缩，两端分别维护自己的 QPACK 编解码器，两端都至多保持一个 encoder stream 和至多一个 decoder stream 和对端保持 QPACK 状态同步。

每一个请求独占一个 request stream，客户端发送请求完成后，必须关闭流的发送端。

## 相关规范

[rfc9000](https://datatracker.ietf.org/doc/html/rfc9000)

[rfc9204](https://datatracker.ietf.org/doc/html/rfc9204)

‍
