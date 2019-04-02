import asyncio
import random
from datetime import datetime
import json


def json_serial(obj):
    if isinstance(obj, datetime):
        return obj.isoformat()
    raise TypeError("Type %s not serializable" % type(obj))


class Storage(object):
    def __init__(self):
        self.total = {'visitors': 0, 'started_at': datetime.now()}
        self.storage = {}

    def new_connection(self, target):
        item = self.storage.get(target)
        if not item:
            item = {'added_at': datetime.now(), 'attempts': 0}
            self.total['visitors'] += 1
        item['attempts'] += 1
        item['reset_at'] = None
        item['is_connected'] = True
        item['last_activity'] = datetime.now()
        self.storage[target] = item

    def close_connection(self, target):
        assert self.storage.get(target)
        item = self.storage[target]
        item['reset_at'] = datetime.now()
        item['is_connected'] = False
        self.storage[target] = item

    def to_json(self):
        self.total['server_uptime'] = (self.total['started_at'] - datetime.now()).seconds
        for target in self.storage:
            current_target = self.storage[target]
            if not current_target['is_connected']:
                continue
            self.storage[target]['target_uptime'] = (current_target['added_at'] - datetime.now()).seconds
        return {
            'total': self.total,
            'detail': self.storage
        }

    def __str__(self):
        return str(self.storage)

    def __getitem__(self, item):
        return self.storage.get(item)


async def statistic_handler(_reader, writer):
    data = json.dumps(storage.to_json(), default=json_serial).encode()
    writer.writelines([
        b'HTTP/1.1 200 OK\r\n',
        b'Connection: close\r\n',
        b'Content-Type: application/json; charset=utf-8\r\n',
        b'Content-Length: %d\r\n' % len(data),
        b'\r\n',
        data,
        b'\r\n'
    ])
    await writer.drain()
    writer.close()


async def tarpit_handler(_reader, writer):
    writer.write(b'HTTP/1.1 200 OK\r\n')
    client_ip, client_port = writer.get_extra_info('peername')
    storage.new_connection(client_ip)
    try:
        while True:
            await asyncio.sleep(5)
            header = random.randint(0, 2 ** 32)
            value = random.randint(0, 2 ** 32)
            writer.write(b'X-%x: %x\r\n' % (header, value))
            await writer.drain()
    except ConnectionResetError:
        storage.close_connection(client_ip)


def run_server(ip='127.0.0.1', tarpit_port=8888, statistic_port=8080):
    loop = asyncio.get_event_loop()
    router = [
        (tarpit_handler, tarpit_port),
        (statistic_handler, statistic_port)
    ]
    for handler, port in router:
        server = asyncio.start_server(handler, ip, port, loop=loop)
        loop.run_until_complete(server)
    print(f'Tarpit started on {ip}:{tarpit_port}')
    print(f'Statistic started on {ip}:{statistic_port}')
    loop.run_forever()


if __name__ == '__main__':
    storage = Storage()
    run_server()
