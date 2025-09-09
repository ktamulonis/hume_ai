# hume_ai

Ruby client for [Hume AI](https://hume.ai) TTS + EVI APIs. This gem  provides client: simple `Client`, resource classes (`Voices`, `TTS`), and helpers for streaming and WebSocket integration.

---

## Installation

Add to your Gemfile:

```ruby
gem "hume_ai"
```

Then run:

```bash
bundle install
```

Or install directly:

```bash
gem install hume_ai
```

---

## Configuration

If you use Rails, add an initializer at `config/initializers/hume_ai.rb`
# Rails initializer example
```ruby
HumeAI.configure do |config|
  config.api_key    = Rails.application.credentials.dig(:hume_ai, :api_key) || ENV["HUME_API_KEY"]
  config.secret_key = Rails.application.credentials.dig(:hume_ai, :secret_key)
end
```

```ruby
HumeAI.configure do |config|
  config.api_key    = Rails.application.credentials.dig(:hume_ai, :api_key)
  config.secret_key = Rails.application.credentials.dig(:hume_ai, :secret_key)
end
```

The gem will use `HumeAI.configuration.api_key` when available. Otherwise it falls back to `ENV["HUME_API_KEY"]`.

---

## Quick start

```ruby
require "hume_ai"

# Use global configuration (initializer) or call client explicitly
client = HumeAI.client

# Resources
voices = HumeAI::Voices.new(client)
tts    = HumeAI::TTS.new(client)

# List available voices (Hume library or your custom voices)
voices.list(provider: "HUME_AI")

# Create/save a custom voice from a prior generation
voices.create(generation_id: "795c949a-...", name: "My Custom Voice")

# Delete a saved custom voice by name
voices.delete(name: "My Custom Voice")
```

---

## Selecting a voice

When requesting TTS output, include a `voice` object inside each utterance. Examples:

```ruby
utterance = {
  text: "Hello from Hume",
  voice: { name: "Male English Actor", provider: "HUME_AI" }
}

# or a custom voice you've saved
utterance_custom = {
  text: "Hi again",
  voice: { name: "My Custom Voice", provider: "CUSTOM_VOICE" }
}

resp = tts.synthesize_json(utterances: [utterance], format: { type: "mp3" })
```

Use `voices.list(provider: "HUME_AI")` to inspect library voices and `voices.list(provider: "CUSTOM_VOICE")` for your saved voices.

---

## Getting audio back (file vs JSON)

### JSON (base64 in response)

```ruby
resp = tts.synthesize_json(
  utterances: [utterance],
  format: { type: "mp3" }
)
base64_audio = resp.dig("generations", 0, "audio")
File.write("out.mp3", Base64.decode64(base64_audio))
```

### File endpoint (binary response)

```ruby
file = tts.synthesize_file(
  utterances: [utterance],
  format: { type: "mp3" }
)
FileUtils.cp(file.path, "out_file.mp3")
```

### Streaming file endpoint (write chunks to disk)

```ruby
File.open("stream.mp3", "wb") do |f|
  tts.stream_file(
    utterances: [utterance],
    format: { type: "mp3" }
  ) do |chunk|
    f.write(chunk)
  end
end
```

---

## Streaming JSON (server-sent JSON objects)

```ruby
tts.stream_json(
  utterances: [utterance],
  format: { type: "mp3" }
) do |chunk|
  # chunk is a Hash with keys like :generation_id, :snippet_id, :audio, :audio_format, :is_last_chunk
  p chunk
  if chunk[:audio]
    # decode snippet-level audio
    raw = Base64.decode64(chunk[:audio])
    File.open("snippet_#{chunk[:snippet_id]}.mp3", "wb") { |f| f.write(raw) }
  end
end
```

---

## Low-latency WebSocket input (TTSStream)

Use `HumeAI::TTSStream` if you need to push text incrementally and receive chunked output over a WebSocket connection.

```ruby
ws = HumeAI::TTSStream.new(api_key: HumeAI.configuration.api_key, format_type: "mp3").connect!
ws.on_chunk { |data| p data }
ws.on_error { |e| warn e }

# send a snippet
ws.send_input(text: "Hello world", voice: { name: "Male English Actor", provider: "HUME_AI" })
# flush to force generation
ws.flush!
# close when done
ws.close!
```

---

## EVI Chat (WebSocket)

EVI (Empathic Voice Interface) accepts both **text** and **audio** messages. The gem includes conveniences to send base64 audio buffers or a local file directly.

```ruby
chat = HumeAI::EVIChat.new(api_key: HumeAI.configuration.api_key, verbose_transcription: true).connect!

# Receive events
chat.on_event { |evt| p evt }
chat.on_error { |err| warn err }

# Send text user input
chat.send_user_input(text: "Hello EVI")

# Send an audio file (convenience helper) — this encodes and sends base64 for you
chat.send_audio_file("path/to/your_input.wav")

# Or encode manually and send
b64 = Base64.strict_encode64(File.binread("path/to/your_input.wav"))
chat.send_audio_input(b64)

# Close when finished
chat.close!
```

Notes:
- `send_audio_file(path)` handles base64 encoding and dispatch for you.
- The websocket yields rich event objects—inspect them to see `UserMessage`, `AssistantMessage`, `AudioOutput`, `ChatMetadata`, and more as specified by Hume.

---

## Error handling

HTTP non-2xx responses raise `HumeAI::APIError` (has `.status` and `.body`). WebSocket errors are surfaced via the `on_error` callback.

---

## Rails tips

- Put credentials in `config/credentials.yml.enc` and reference them in `config/initializers/hume_ai.rb` as shown above.
- Consider wrapping streaming interactions inside background jobs if you intend to persist generated audio or long-running sessions.

---

## Development & Contributing

PRs welcome. Recommended development flow:

```bash
git clone <repo>
bundle install
bundle exec rake spec
```

Add specs for new helpers (particularly for Base64 handling and WebSocket behavior). Mock external requests with `webmock` or `vcr` in CI.

---

## License

MIT

# File: LICENSE
MIT License

Copyright (c) 2025 hackliteracy

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

# hume_ai
