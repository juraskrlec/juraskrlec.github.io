---
layout: post
title:  "Audio Engine/JSWaveform"
date:   2024-07-18
categories: image
---

I haven't touched Swift and SwiftUI for a while, but watching the WWDC24 videos reignited my excitement to dive deeper into Swift and SwiftUI. I’m eager to learn more about them and integrate them into all my future projects in some form. Having worked with Objective-C++ for over a decade, I'm particularly interested in understanding the differences and learning how to tackle problems using Swift, especially with the new Swift Concurrency features.

The addition of C++ interoperability in Swift (a topic for another post) is fantastic for transitioning legacy Objective-C++ projects to Swift. Additionally, there’s a specific area I’ve been wanting to explore for a while: audio development on iOS. So, I decided to combine my interest in Swift with AVAudioEngine, and that’s how the JSWaveform was born. You can check out the Swift package on my [Github](https://github.com/juraskrlec/JSWaveform).

In this post, I will delve into the intricacies of an Audio Engine written in Swift, leveraging the power of Apple's AVFoundation framework. We'll explore how to manage audio playback, apply pitch effects, and handle asynchronous audio buffering in a robust and efficient manner.

**What is AVAudioEngine?**

AVAudioEngine is an advanced audio framework that lets you build intricate audio processing chains using a graph of audio nodes. Each node performs a specific function, such as playing an audio file, applying effects, or mixing multiple audio streams. The modularity of AVAudioEngine allows you to customize and extend audio processing according to your application's needs.

**Key Components of AVAudioEngine**

Understanding the key components of AVAudioEngine is essential for leveraging its full potential:

1. Nodes: Nodes are the fundamental units in AVAudioEngine. They perform various audio tasks and can be connected to form complex audio processing graphs.

    * AVAudioInputNode: Captures audio from the microphone.
    * AVAudioOutputNode: Outputs processed audio to the device’s speakers or headphones.
    * AVAudioPlayerNode: Plays audio files or buffers.
    * AVAudioUnitEffect: Applies audio effects such as reverb or delay.
    * Engine: The AVAudioEngine class manages the audio nodes and the connections between them, ensuring synchronized processing.

2. AVAudioEngine: The core class that orchestrates the entire audio processing flow.
3. Buffers: Buffers temporarily store audio data during processing, enabling smooth transitions and real-time manipulation.

**JSWaveform**

JSWaveform is a Swift Package that has native interfaces consisting of audio engine and pure animatable SwiftUI components in iOS, iPadOS and visionOS.

JSWaveform provides native Swift and SwiftUI components. For now, it has 2 major SwiftUI views:

* AudioPlayerView - renders audio player which consits of play/pause button, downsampled waveform and time pitch effect button.
* AudioVisualizerView - renders audio visualizer which animates AudioVisualizerShape based on audio amplitudes. 

All the code for manipulating audio signals follows `JSWaveform`.

**Setting Up the Audio Engine**

We define an AudioEngine actor to encapsulate the functionality:

```swift
actor AudioEngine {
    
    private let avAudioEngine = AVAudioEngine()
    private let audioPlayer = AVAudioPlayerNode()
    private let audioTimePitch = AVAudioUnitTimePitch()
    private var audioBuffer: AVAudioPCMBuffer?
    private var asyncBufferStream: AsyncStream<AVAudioPCMBuffer>?
    private var continuation: AsyncStream<AVAudioPCMBuffer>.Continuation?
    
    private let logger = Logger(subsystem: "JSWaveform.AudioEngine", category: "Engine")
    
    enum AudioEngineError: Error {
        case bufferRetrieveError
    }
    
    init() {
        avAudioEngine.attach(audioPlayer)
        avAudioEngine.attach(audioTimePitch)
    }
}
```

The setup function connects the various nodes and prepares the engine for playback:

```swift
nonisolated func setup() {
    let output = avAudioEngine.outputNode
    let mainMixer = avAudioEngine.mainMixerNode

    avAudioEngine.connect(audioPlayer, to: audioTimePitch, format: nil)
    avAudioEngine.connect(audioTimePitch, to: mainMixer, format: nil)
    avAudioEngine.connect(mainMixer, to: output, format: nil)
    avAudioEngine.prepare()
}
```

We prepare an asynchronous audio buffer stream to handle audio data efficiently:

```swift
func prepareBuffer() {
    asyncBufferStream = AsyncStream { continuation in
         self.continuation = continuation
     }
    
    audioPlayer.installTap(onBus: 0, bufferSize: 256, format: nil) { buffer, _ in
        self.continuation?.yield(buffer)
    }
}

func getBuffer() -> AsyncStream<AVAudioPCMBuffer>? {
    return asyncBufferStream
}
```

The `audioPlayer.installTap(onBus:bufferSize:format:block:)` method is an essential part of the AudioEngine class, which taps into the audio signal coming from the audioPlayer node, allowing you to process or analyze audio data in real-time. Let's break down the components and significance of this method call.

The `installTap` method in AVAudioNode allows you to create a tap on a specified bus. This tap intercepts the audio data flowing through the node, providing you with a buffer of audio samples that you can use for various purposes like visualization, analysis, or further processing.

Parameters of the `installTap` method:

* onBus: The bus number you want to tap into. For most audio nodes, bus 0 is the default output bus.
* bufferSize: The number of audio samples per channel in each buffer that the tap handler receives. In this case, it is set to 256.
* format: The audio format of the tap. Passing nil means the format of the tap will be the format of the node's output bus.
* block: A block (closure) that processes the audio buffers. This block is called with each buffer of audio data.

The block (closure) provided to `installTap` is where the audio processing happens:

* buffer: This is the AVAudioPCMBuffer containing the audio samples tapped from the audio node.
* (AVAudioTime): This parameter provides the time at which the buffer's audio data is rendered. In this case, it's ignored.

The closure yields the buffer to the `AsyncStream` continuation. This line sends the tapped audio buffer to the AsyncStream, making it available for asynchronous processing. This approach leverages Swift's concurrency model to handle audio data efficiently, ensuring smooth real-time performance.

**Why Buffer Size 256?**

The buffer size of 256 frames (samples per channel) is chosen for several reasons:

* Latency vs. Performance Trade-Off: A smaller buffer size means lower latency, which is critical for real-time audio applications. However, smaller buffers require the CPU to handle more frequent interrupts, increasing the processing load. A buffer size of 256 is a balanced choice, providing relatively low latency without overwhelming the CPU.

* Consistency with Audio Standards: The buffer size of 256 frames is commonly used in audio processing as it aligns well with typical audio sample rates (like 44.1kHz or 48kHz), resulting in efficient processing and compatibility with audio hardware and software standards.

* Smooth Real-Time Processing: For real-time applications like audio playback, effects, and recording, using a buffer size of 256 ensures smooth operation. It allows the audio engine to deliver consistent audio data with minimal delay, enhancing the user experience in interactive audio applications.

**Audio Processing**

For animations, we need to processes an audio buffer to calculate the power levels (both average and peak) for each channel in the buffer.

```swift
func process(buffer: AVAudioPCMBuffer) {
    var powerLevels = [PowerLevels]()
    let channelCount = Int(buffer.format.channelCount)
    let length = vDSP_Length(buffer.frameLength)

    if let floatData = buffer.floatChannelData {
        for channel in 0..<channelCount {
            powerLevels.append(calculatePowers(data: floatData[channel], strideFrames: buffer.stride, length: length))
        }
    } else if let int16Data = buffer.int16ChannelData {
        for channel in 0..<channelCount {
            var floatChannelData: [Float] = Array(repeating: Float(0.0), count: Int(buffer.frameLength))
            vDSP_vflt16(int16Data[channel], buffer.stride, &floatChannelData, buffer.stride, length)
            var scalar = Float(INT16_MAX)
            vDSP_vsdiv(floatChannelData, buffer.stride, &scalar, &floatChannelData, buffer.stride, length)

            powerLevels.append(calculatePowers(data: floatChannelData, strideFrames: buffer.stride, length: length))
        }
    } else if let int32Data = buffer.int32ChannelData {
        for channel in 0..<channelCount {
            var floatChannelData: [Float] = Array(repeating: Float(0.0), count: Int(buffer.frameLength))
            vDSP_vflt32(int32Data[channel], buffer.stride, &floatChannelData, buffer.stride, length)
            var scalar = Float(INT32_MAX)
            vDSP_vsdiv(floatChannelData, buffer.stride, &scalar, &floatChannelData, buffer.stride, length)

            powerLevels.append(calculatePowers(data: floatChannelData, strideFrames: buffer.stride, length: length))
        }
    }
    self.values = powerLevels
}
```

If the buffer contains floating-point data, each channel's data is processed directly. If the buffer contains 16-bit integer data, it is converted to floating-point data. The conversion is performed using the Accelerate framework's `vDSP_vflt16` function. The data is then normalized by dividing by `INT16_MAX`. The same calculations are done if the channel contains 32-bit integer data.

```swift
private func calculatePowers(data: UnsafePointer<Float>, strideFrames: Int, length: vDSP_Length) -> PowerLevels {
    var max: Float = 0.0
    vDSP_maxv(data, strideFrames, &max, length)
    if max < kMinLevel {
        max = kMinLevel
    }

    var rms: Float = 0.0
    vDSP_rmsqv(data, strideFrames, &rms, length)
    if rms < kMinLevel {
        rms = kMinLevel
    }

    return PowerLevels(average: 20.0 * log10(rms), peak: 20.0 * log10(max))
}
```

This function calculates the Root Mean Square (RMS) and peak power levels from the audio data. The function returns these values encapsulated in a `PowerLevels` struct. Here is a detailed explanation of each component:

1. RMS (Root Mean Square):

    * The RMS value is a measure of the average power of the audio signal.
    * It provides a meaningful average level of the waveform's power.
    * The vDSP_rmsqv function from the Accelerate framework computes the RMS value of the audio data.
    * If the RMS value is less than kMinLevel, it is set to kMinLevel to avoid taking the logarithm of zero. kMinLevel is defined as kMinLevel: Float = 0.000_000_01 // -160 dB.

2. Peak Level:

    * The peak value represents the maximum power level of the audio signal.
    * It indicates the highest amplitude in the audio data.
    * The vDSP_maxv function from the Accelerate framework finds the maximum value in the audio data.

3. Decibel Conversion:

    * Audio levels are often represented in decibels (dB) because the human ear perceives sound logarithmically.
    * The conversion from linear power values to decibels is done using the logarithm function:

    ```swift
    let average = 20.0 * log10(rms)
    let peak = 20.0 * log10(max)
    ```

    * Multiplying by 20 converts the logarithm of the power ratio into decibels.

**AudioVisualizer**

Now when we have some data, we need to use it. In our Model, we process audio buffer and update our observable `amplitudes`. To have smooth animations based on screen refresh rate, we need to use `CADisplayLink`. 

```swift
public func processAudio() async {
    guard let bufferStream = await audioEngine.getBuffer() else { return }

    for await buffer in bufferStream {
        if isPlaying {
            audioProcessing.process(buffer: buffer)
        } else {
            audioProcessing.processSilence()
        }
    }
}
```

```swift
// MARK: DisplayLink update method
override func update() {
    
    guard let levels = audioLevel.levelProvider?.levels else {
        return
    }
    
    level = CGFloat(levels.level)
    peakLevel = CGFloat(levels.peakLevel)
    
    var targetAmplitudes:[Double]

    // Example calculation
    let halfCount = amplitudes.count / 2
    let newAmplitudes = (0..<halfCount).map { index in
        let normalizedIndex = Double(index) / Double(halfCount - 1)
        let scalingFactor = 1.0 + normalizedIndex * 1.5
        let amplitude = level * (1.0 - normalizedIndex) + peakLevel * normalizedIndex
        return min(1.0, max(0.0, pow(amplitude * scalingFactor, 1.2)))
    }
    targetAmplitudes = newAmplitudes + newAmplitudes.reversed()

    if peakLevel > 0 {
        self.amplitudes = self.amplitudes.enumerated().map { index, current in
            self.lowPassFilter(currentValue: current, targetValue: targetAmplitudes[index])
        }
    }
}

private func lowPassFilter(currentValue: Double, targetValue: Double, smoothing: Double = 0.1) -> Double {
    return currentValue * (1.0 - smoothing) + targetValue * smoothing
}
```

`lowPassFilter` function is a simple yet effective method for smoothing or filtering a signal. This particular implementation is often used in audio processing, sensor data smoothing, and other applications where you want to reduce noise or fluctuations in a signal. We use it here for smooth animations.

![AudioVisualizer](https://github.com/user-attachments/assets/9b0e44ae-fbc5-4316-9be1-a2b9df3ba61e)

**AudioPlayer**

But what when you want to play audio file, and show a waveform, and skip audio while dragging on waveform like, for example, voice message?

```swift
func loadSamples(forURL url: URL, downsampledTo targetSampleCount: Int) async throws -> [Float] {
    logger.debug("Loading audio samples for URL: \(url), downsampled: \(targetSampleCount)")
    
    let file = try? AVAudioFile(forReading: url)
    guard let format = file?.processingFormat, let length = file?.length else { throw WafevormError.audioFileNotFound }
    
    let buffer = AVAudioPCMBuffer(pcmFormat: format, frameCapacity: AVAudioFrameCount(length))
    try? file?.read(into: buffer!)
    
    guard let floatChannelData = buffer?.floatChannelData else { throw WafevormError.bufferRetrieveError }
    let channelData = floatChannelData.pointee
    
    let sampleCount = Int(length)
    let samplesPerPixel = sampleCount / targetSampleCount
    var downsampledData = [Float]()
    
    for i in 0..<targetSampleCount {
        let start = i * samplesPerPixel
        let end = min((i + 1) * samplesPerPixel, sampleCount)
        let sampleRange = start..<end
        
        let maxSample = sampleRange.map { channelData[$0] }.max() ?? 0
        downsampledData.append(maxSample)
    }
    
    return downsampledData
}

func loadSamples(downsampledTo: Int) async throws -> [Float] {
    let samples = try await waveformLoader.loadSamples(forURL: audioURL, downsampledTo: downsampledTo)
    normalizedSamples = normalizeWaveformData(samples)
    return normalizedSamples
}

private func normalizeWaveformData(_ data: [Float]) -> [Float] {
    guard let maxSample = data.max() else { return [] }
    return data.map { $0 / maxSample }
}
```

These functions loads audio data from a file, downsamples it to a specified number of samples, normalize the data and returns the downsampled data. The process involves reading the audio file, creating a buffer to hold the data, retrieving the data, calculating the appropriate downsampling parameters, and then iterating through the data to extract and store the downsampled samples. The use of the maximum value within each downsampled segment helps preserve the peaks in the audio data. You'll get something like this:

![AudioPlayer](https://github.com/user-attachments/assets/4c7ada40-4a9d-44dd-ae11-89a9d5b92cab)

**Conclusion**

AVAudioEngine is a versatile and powerful tool that opens up a world of possibilities for iOS developers. By mastering its components and features, you can create rich and immersive audio experiences in your applications. Whether you are developing a simple audio player or a sophisticated audio processing app, AVAudioEngine provides the tools you need to bring your audio visions to life. Happy coding!
