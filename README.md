# BT-Speaker

A simple Bluetooth Speaker Daemon designed for the Raspberry Pi 3.

## Installation

Quick Installation for Raspbian:

```bash
sudo -i
bash <(curl -s https://raw.githubusercontent.com/lukasjapan/bt-speaker/master/install.sh)
```

For details refer to the comments in the [install script](https://github.com/lukasjapan/bt-speaker/blob/master/install.sh).

Depending on your application, you might also want to send all audio to the headphone jack.
This can be done by `raspi-config`:

`Advanced Options` -> `Audio` -> `Force 3.5mm ('headphone') jack`

## Usage

The BT-Speaker daemon does not behave like a typical bluetooth device.
Once a client disconnects, the speaker will immediately allow other clients to connect.
This means that the quickest device may claim the speaker and no real bluetooth pairing occurs.
The bright side of this logic is that no button for unpairing is needed.

The speakers name will default to the hostname of your Raspberry Pi.
BT-Speaker does not manage this value.
You are advised to change the hostname according to your needs.

## Details of Implementation

The BT-Speaker daemon has been written in Python and works with Bluez5.
It talks to the Bluez daemon via the [Bluez DBUS interface](https://git.kernel.org/cgit/bluetooth/bluez.git/tree/doc).

### Bluetooth profiles

BT-Speaker will register itself as an [A2DP](https://en.wikipedia.org/wiki/List_of_Bluetooth_profiles#Advanced_Audio_Distribution_Profile_.28A2DP.29) capable device and route the received audio fully decoded to ALSAs `aplay` command.

Changes in volume are detected via messages from the [AVRCP](https://en.wikipedia.org/wiki/List_of_Bluetooth_profiles#Audio.2FVideo_Remote_Control_Profile_.28AVRCP.29) profile and are applied directly to the ALSA master volume.

### Partial Bluez5 port of BT-Manager

The great [BT-Manager](https://github.com/liamw9534/bt-manager) library does (currently) only work with Bluez4.
Changes in the Bluez DBUS API [from version 4 to 5](http://www.bluez.org/bluez-5-api-introduction-and-porting-guide/) were huge and fully porting BT-Manager would have been a too heavy task.
So instead, I extracted all relevant parts and ported them to Bluez5 as good as I could.
Documentation and probably lots of other parts there have yet to be adjusted, so refer to [that code](bt_manager) with caution.

### About the audio stream

The following describes some internals of the audio stream that is transferred via bluetooth.

#### Format

The [Light of Dawn blog](http://www.lightofdawn.org/blog/?viewCat=Bluetooth) describes the format very accurate:

> As it turns out, the audio data is compressed with [SBC codec](http://en.wikipedia.org/wiki/SBC_%28codec%29).
> But I can't just use "sbcdec" tool from SBC package to decode it, as the audio data is encapsulated in A2DP packets, not naked SBC-compressed audio data.
> A2DP packets are RTP packets (referenced by [A2DP specification](https://www.bluetooth.org/en-us/specification/adopted-specifications), and detailed in [this IETF draft](http://tools.ietf.org/html/draft-ietf-payload-rtp-sbc-04)) containing A2DP Media Payload.
> We need to extract the SBC audio data, pass it through SBC decompressor, and only then we get raw audio data that can be sent to ALSA.

Unfortunately there is no media player (or at least I didn't find any) that could handle this 'SBC in RTP' format natively.
However, BT-Manager already provided a C library that takes care of the decoding process.
The decoded output is raw audio data in CD format (16 bit little endian, 44100Hz, stereo) and can be piped to ALSA as mentioned in the blog.

#### Decoding

The C library for the decoding process is located in the [codecs](codecs) folder.
Its functions are called via [Python CFFI](http://cffi.readthedocs.io/en/latest/).
BT-Speaker provides the binary for ARM already, so there is no need to compile the codec manually.

However, if you need to do so for some reason, please be aware that the Makefile has been adjusted by the following:

1. The default `PLATFORM` setting has been changed to `armv6`
1. The `-O3` flag has been added

#### Output

The decoded raw audio data can be piped to any command.
It defaults to `aplay -f cd -` which will send the data straight to the sound card.
You can change the output command at [bt_speaker.py](bt_speaker.py#L20) if you want to further process the stream.
