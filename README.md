# SMS MQTT gateway

A small service that turns an MQTT message into a real SMS, sent through a GSM modem.

Publish this to `sms/outgoing`:

```json
{ "phone_number": "+213xxxxxxxxx", "message": "Your appointment is tomorrow at 9." }
```

and a phone receives a text, over the cellular network, with no internet involved on the
delivery side.

## Why it exists

This is the edge component of **Taqwim DZ** ([`notify-me-mobile`](https://github.com/yazi-mellissa/notify-me-mobile)),
a reminder application for users in Algeria. The problem it solves is that push notifications and
instant messaging both assume the recipient has data coverage at the moment the reminder
matters. In much of the country that assumption does not hold.

SMS does hold, because it rides on the signalling channel rather than on data. So the
architecture puts a Raspberry Pi with a GSM module on the edge: the application publishes a
reminder to a broker, this service picks it up and dials out over the modem. The rest of the
system never needs to know how delivery happened.

## How it works

```
application  --MQTT-->  mosquitto  --subscribe-->  this service  --Gammu-->  GSM modem  -->  SMS
                                                                   /dev/serial0
```

| Piece | What it does |
|---|---|
| [`messaging_service/main.py`](messaging_service/main.py) | Subscribes to the topic, parses the JSON payload, validates that both fields are present, and hands the message to Gammu |
| [`messaging_service/.gammurc`](messaging_service/.gammurc) | Modem configuration: serial port, baud rate, and the SMSD spool directories |
| [`mosquitto_config/mosquitto.conf`](mosquitto_config/mosquitto.conf) | Broker listener |
| [`docker-compose.yml`](docker-compose.yml) | Brings up the broker and the service, passing the host's serial device through to the container |

The MQTT client uses `CallbackAPIVersion.VERSION2` with MQTT v5, so it works with paho-mqtt 2.x
rather than the 1.x API that most examples still show.

## Running it

```bash
cp messaging_service/.env.example messaging_service/.env
docker compose up -d
docker compose logs -f messaging-service
```

Sending a test message, with the broker running:

```bash
mosquitto_pub -h localhost -t sms/outgoing \
  -m '{"phone_number":"+213xxxxxxxxx","message":"test"}'
```

### Requirements

- A GSM modem that Gammu recognises. Check it with `gammu --identify` before anything else:
  if that fails, nothing downstream will work.
- A SIM with credit, and the serial device exposed to the host (here `/dev/serial0`, the
  Raspberry Pi UART). Change the `devices:` entry in `docker-compose.yml` for a different board
  or a USB dongle.
- Docker and Docker Compose.

### Configuration

| Variable | Default | Meaning |
|---|---|---|
| `MQTT_BROKER` | `mqtt-broker` | Broker hostname, the compose service name by default |
| `MQTT_TOPIC` | `sms/outgoing` | Topic to subscribe to |

## What this is not ready for

The service was built for a closed deployment: one Pi, one broker, on a private network. Three
things would have to change before it faced anything else, and they are listed here rather than
left for someone to discover.

- **The broker accepts anyone.** `allow_anonymous true` with no TLS. On a reachable network that
  means any client can publish to `sms/outgoing`, and every message published costs real money
  and goes to a real phone. Authentication and TLS come first.
- **The container is privileged.** `privileged: true` is the blunt way to reach the serial
  device. Granting the specific device and the `dialout` group is the narrow way.
- **A failed send is lost.** If the modem is busy or the SIM is out of credit, the error is
  logged and the message disappears. There is no retry, no dead-letter topic, and no delivery
  report read back from the modem, so the publisher never learns whether its SMS arrived. For a
  reminder system that is the gap that matters most.

## Credits

Built with **Khaled Benmachiche**. The original commit history is in his repository,
[`khaledbenmachiche/taqwim-messaging-gateway`](https://github.com/khaledbenmachiche/taqwim-messaging-gateway).

This repository carries the same code with one change: the Dockerfile no longer copies `.env`
into the image. Configuration is passed through Compose instead, so a built image holds no
settings and `.env` stays out of version control.

## License

MIT, see [LICENSE](LICENSE).
