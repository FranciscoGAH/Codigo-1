# Codigo-1
### Código para la activación del modulo BLE de la Raspberry Pi Pico 2 W ###

import bluetooth
from micropython import const

# EVENTOS BLE
_IRQ_CENTRAL_CONNECT = const(1)
_IRQ_CENTRAL_DISCONNECT = const(2)
_IRQ_GATTS_WRITE = const(3)

# UUID UART BLE
_UART_UUID = bluetooth.UUID("6E400001-B5A3-F393-E0A9-E50E24DCCA9E")
_UART_TX  = bluetooth.UUID("6E400003-B5A3-F393-E0A9-E50E24DCCA9E")
_UART_RX  = bluetooth.UUID("6E400002-B5A3-F393-E0A9-E50E24DCCA9E")

_UART_TX_CHAR = (_UART_TX, bluetooth.FLAG_NOTIFY)
_UART_RX_CHAR = (_UART_RX, bluetooth.FLAG_WRITE)

_UART_SERVICE = (_UART_UUID, (_UART_TX_CHAR, _UART_RX_CHAR))

# BLE SIMPLE PERIPHERAL
class BLESimplePeripheral:
    def __init__(self, ble, name="BLE_Device"):
        self._ble = ble
        self._ble.active(True)
        self._ble.irq(self._irq)

        ((self._tx_handle, self._rx_handle),) = self._ble.gatts_register_services(
            (_UART_SERVICE,)
        )

        self._connections = set()
        self._payload = self._advertising_payload(name)
        self._advertise()

    def _irq(self, event, data):
        if event == _IRQ_CENTRAL_CONNECT:
            conn_handle, _, _ = data
            self._connections.add(conn_handle)

        elif event == _IRQ_CENTRAL_DISCONNECT:
            conn_handle, _, _ = data
            self._connections.remove(conn_handle)
            self._advertise()

        elif event == _IRQ_GATTS_WRITE:
            conn_handle, value_handle = data
            if value_handle == self._rx_handle:
                self.on_write(self._ble.gatts_read(self._rx_handle))

    def send(self, data):
        for conn_handle in self._connections:
            self._ble.gatts_notify(conn_handle, self._tx_handle, data)

    def on_write(self, data):
        pass 

    def _advertise(self, interval_us=100_000):
        self._ble.gap_advertise(interval_us, self._payload)

    def _advertising_payload(self, name):
        payload = bytearray(b'\x02\x01\x06')
        payload += bytes((len(name) + 1, 0x09)) + name.encode()
        return payload

