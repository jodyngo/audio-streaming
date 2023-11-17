# audio-streaming

#include <iostream>
#include <alsa/asoundlib.h>

#define BUFFER_SIZE 4096

int main() {
    int err;
    char *buffer;
    snd_pcm_t *capture_handle;
    snd_pcm_hw_params_t *hw_params;

    // Open the capture device (microphone)
    err = snd_pcm_open(&capture_handle, "default", SND_PCM_STREAM_CAPTURE, 0);
    if (err < 0) {
        std::cout << "Cannot open capture device: " << snd_strerror(err) << std::endl;
        return 1;
    }

    // Set hardware parameters for capturing
    snd_pcm_hw_params_malloc(&hw_params);
    snd_pcm_hw_params_any(capture_handle, hw_params);
    snd_pcm_hw_params_set_access(capture_handle, hw_params, SND_PCM_ACCESS_RW_INTERLEAVED);
    snd_pcm_hw_params_set_format(capture_handle, hw_params, SND_PCM_FORMAT_S16_LE);
    snd_pcm_hw_params_set_channels(capture_handle, hw_params, 2);  // Stereo audio
    unsigned int sample_rate = 44100;  // Sample rate (adjust as needed)
    snd_pcm_hw_params_set_rate_near(capture_handle, hw_params, &sample_rate, 0);
    snd_pcm_hw_params(capture_handle, hw_params);
    snd_pcm_hw_params_free(hw_params);

    // Allocate buffer to hold captured audio data
    buffer = (char *)malloc(BUFFER_SIZE);

    // Open the remote playback device (speaker)
    snd_pcm_t *playback_handle;
    err = snd_pcm_open(&playback_handle, "default", SND_PCM_STREAM_PLAYBACK, 0);
    if (err < 0) {
        std::cout << "Cannot open playback device: " << snd_strerror(err) << std::endl;
        return 1;
    }

    // Set hardware parameters for playback
    snd_pcm_hw_params_malloc(&hw_params);
    snd_pcm_hw_params_any(playback_handle, hw_params);
    snd_pcm_hw_params_set_access(playback_handle, hw_params, SND_PCM_ACCESS_RW_INTERLEAVED);
    snd_pcm_hw_params_set_format(playback_handle, hw_params, SND_PCM_FORMAT_S16_LE);
    snd_pcm_hw_params_set_channels(playback_handle, hw_params, 2);  // Stereo audio
    snd_pcm_hw_params_set_rate_near(playback_handle, hw_params, &sample_rate, 0);
    snd_pcm_hw_params(playback_handle, hw_params);
    snd_pcm_hw_params_free(hw_params);

    // Start capturing and playback loop
    while (true) {
        // Capture audio from the microphone
        err = snd_pcm_readi(capture_handle, buffer, BUFFER_SIZE / 4);
        if (err != BUFFER_SIZE / 4) {
            std::cout << "Capture error: " << snd_strerror(err) << std::endl;
        }

        // Play captured audio to the speaker
        err = snd_pcm_writei(playback_handle, buffer, BUFFER_SIZE / 4);
        if (err != BUFFER_SIZE / 4) {
            std::cout << "Playback error: " << snd_strerror(err) << std::endl;
        }
    }

    // Clean up and close the audio devices
    free(buffer);
    snd_pcm_close(capture_handle);
    snd_pcm_close(playback_handle);

    return 0;
}

Write C++ code for below scenario in linux: Client side holds microphone in Linux, server hold the remote camera and speaker. Client receive video from the remote camera and display in client. client sends audio to remote speaker.

#include <iostream>
#include <gst/gst.h>
#include <gst/app/gstappsink.h>
#include <alsa/asoundlib.h>

#define BUFFER_SIZE 4096

// Structure to hold the GStreamer pipeline elements and data
struct PipelineData {
    GstElement* pipeline;
    GstElement* videoSink;
    GstElement* audioSink;
};

// GStreamer bus callback function to handle messages
static gboolean busCallback(GstBus* bus, GstMessage* message, gpointer data) {
    switch (GST_MESSAGE_TYPE(message)) {
        case GST_MESSAGE_EOS:
            g_print("End of stream\n");
            break;
        case GST_MESSAGE_ERROR: {
            gchar* debug;
            GError* error;
            gst_message_parse_error(message, &error, &debug);
            g_printerr("Error: %s\n", error->message);
            g_error_free(error);
            g_free(debug);
            break;
        }
        default:
            break;
    }

    return TRUE;
}

// GStreamer app sink callback function to process video samples
static GstFlowReturn videoSampleCallback(GstAppSink* sink, gpointer data) {
    GstSample* sample = gst_app_sink_pull_sample(sink);
    if (sample != NULL) {
        GstBuffer* buffer = gst_sample_get_buffer(sample);
        GstMapInfo map;
        gst_buffer_map(buffer, &map, GST_MAP_READ);

        // Process video frame data in 'map.data' with 'map.size'

        gst_buffer_unmap(buffer, &map);
        gst_sample_unref(sample);
    }

    return GST_FLOW_OK;
}

int main() {
    // Initialize GStreamer
    gst_init(NULL, NULL);

    // Create the pipeline
    PipelineData pipelineData;
    pipelineData.pipeline = gst_pipeline_new("client-pipeline");

    // Create the video source element
    GstElement* videoSource = gst_element_factory_make("tcpclientsrc", "video-source");
    g_object_set(videoSource, "host", "SERVER_IP_ADDRESS", NULL);  // Replace with the actual server IP address
    g_object_set(videoSource, "port", 5000, NULL);  // Replace with the actual server port
    gst_bin_add(GST_BIN(pipelineData.pipeline), videoSource);

    // Create the video decoder element
    GstElement* videoDecoder = gst_element_factory_make("decodebin", "video-decoder");
    gst_bin_add(GST_BIN(pipelineData.pipeline), videoDecoder);
    gst_element_link(videoSource, videoDecoder);

    // Create the video sink element (app sink)
    pipelineData.videoSink = gst_element_factory_make("appsink", "video-sink");
    GstAppSink* videoAppSink = GST_APP_SINK(pipelineData.videoSink);
    gst_bin_add(GST_BIN(pipelineData.pipeline), pipelineData.videoSink);
    gst_element_link(videoDecoder, pipelineData.videoSink);
    gst_app_sink_set_emit_signals(videoAppSink, TRUE);
    gst_app_sink_set_drop(videoAppSink, TRUE);
    gst_app_sink_set_max_buffers(videoAppSink, 1);
    GstAppSinkCallbacks videoCallbacks;
    videoCallbacks.eos = NULL;
    videoCallbacks.new_preroll = NULL;
    videoCallbacks.new_sample = videoSampleCallback;
    gst_app_sink_set_callbacks(videoAppSink, &videoCallbacks, NULL, NULL);

    // Create the audio source element (ALSA microphone)
    GstElement* audioSource = gst_element_factory_make("alsasrc", "audio-source");
    gst_bin_add(GST_BIN(pipelineData.pipeline), audioSource);

    // Create the audio encoder element
    GstElement* audioEncoder = gst_element_factory_make("audioconvert", "audio-encoder");
    gst_bin_add(GST_BIN(pipelineData.pipeline), audioEncoder);
    gst_element_link(audioSource, audioEncoder);

    // Create the audio sink element (ALSA speaker)
    pipelineData.audioSink = gst_element_factory_make("alsasink", "audio-sink");
    gst_bin_add(GST_BIN(pipelineData.pipeline), pipelineData.audioSink);
    gst_element_link(audioEncoder, pipelineData.audioSink);

    // Set up the GStreamer bus to handle messages
    GstBus* bus = gst_element_get_bus(pipelineData.pipeline);
    gst_bus_add_watch(bus, busCallback, NULL);
    gst_object_unref(bus);

    // Start the pipeline
    gst_element_set_state(pipelineData.pipeline, GST_STATE_PLAYING);

    // Open the ALSA microphone device
    snd_pcm_t* captureHandle;
    int err = snd_pcm_open(&captureHandle, "default", SND_PCM_STREAM_CAPTURE, 0);
    if (err < 0) {
        std::cout << "Cannot open capture device: " << snd_strerror(err) << std::endl;
        return 1;
    }

    // Set hardware parameters for capturing
    snd_pcm_hw_params_t* hw_params;
    snd_pcm_hw_params_malloc(&hw_params);
    snd_pcm_hw_params_any(captureHandle, hw_params);
    snd_pcm_hw_params_set_access(captureHandle, hw_params, SND_PCM_ACCESS_RW_INTERLEAVED);
    snd_pcm_hw_params_set_format(captureHandle, hw_params, SND_PCM_FORMAT_S16_LE);
    snd_pcm_hw_params_set_channels(captureHandle, hw_params, 2);  // Stereo audio
    unsigned int sampleRate = 44100;  // Sample rate (adjust as needed)
    snd_pcm_hw_params_set_rate_near(captureHandle, hw_params, &sampleRate, 0);
    snd_pcm_hw_params(captureHandle, hw_params);
    snd_pcm_hw_params_free(hw_params);

    // Allocate buffer to hold captured audio data
    char* buffer = (char*)malloc(BUFFER_SIZE);

    // Open the ALSA speaker device
    snd_pcm_t* playbackHandle;
    err = snd_pcm_open(&playbackHandle, "default", SND_PCM_STREAM_PLAYBACK, 0);
    if (err < 0) {
        std::cout << "Cannot open playback device: " << snd_strerror(err) << std::endl;
        return 1;
    }

    // Set hardware parameters for playback
    snd_pcm_hw_params_malloc(&hw_params);
    snd_pcm_hw_params_any(playbackHandle, hw_params);
    snd_pcm_hw_params_set_access(playbackHandle, hw_params, SND_PCM_ACCESS_RW_INTERLEAVED);
    snd_pcm_hw_params_set_format(playbackHandle, hw_params, SND_PCM_FORMAT_S16_LE);
    snd_pcm_hw_params_set_channels(playbackHandle, hw_params, 2);  // Stereo audio
    snd_pcm_hw_params_set_rate_near(playbackHandle, hw_params, &sampleRate, 0);
    snd_pcm_hw_params(playbackHandle, hw_params);
    snd_pcm_hw_params_free(hw_params);

    // Audio streaming loop
    while (true) {
        // Capture audio from the microphone
        err = snd_pcm_readi(captureHandle, buffer, BUFFER_SIZE / 4);
        if (err != BUFFER_SIZE / 4) {
            std::cout << "Capture error: " << snd_strerror(err) << std::endl;
        }

        // Play captured audio to the speaker
        err = snd_pcm_writei(playbackHandle, buffer, BUFFER_SIZE / 4);
        if (err != BUFFER_SIZE / 4) {
            std::cout << "Playback error: " << snd_strerror(err) << std::endl;
        }
    }

    // Clean up and close the audio devices
    free(buffer);
    snd_pcm_close(captureHandle);
    snd_pcm_close(playbackHandle);

    // Stop the pipeline and release resources
    gst_element_set_state(pipelineData.pipeline, GST_STATE_NULL);
    gst_object_unref(pipelineData.pipeline);

    return 0;
}


 Here's an example of C++ code that demonstrates transmitting audio from a Linux system to a remote speaker over the network using the G.726 algorithm. 

#include <iostream>
#include <fstream>
#include <cstring>
#include <cstdlib>
#include <netinet/in.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#define MAX_BUFFER_SIZE 1024

// Function to read audio data from a file
bool readAudioData(const char* filename, char* buffer, int bufferSize) {
    std::ifstream audioFile(filename, std::ios::binary);
    if (!audioFile) {
        std::cout << "Failed to open audio file." << std::endl;
        return false;
    }

    audioFile.read(buffer, bufferSize);
    if (!audioFile) {
        std::cout << "Failed to read audio data from file." << std::endl;
        return false;
    }

    return true;
}

int main() {
    const char* serverIP = "192.168.0.100";  // Replace with the IP address of the remote speaker
    const int serverPort = 5000;  // Replace with the port number on which the remote speaker is listening
    const char* audioFile = "audio.wav";  // Replace with the path to your audio file

    // Read audio data from file
    char audioBuffer[MAX_BUFFER_SIZE];
    if (!readAudioData(audioFile, audioBuffer, MAX_BUFFER_SIZE)) {
        std::cout << "Failed to read audio data." << std::endl;
        return 1;
    }

    // Create a UDP socket
    int socketFd = socket(AF_INET, SOCK_DGRAM, 0);
    if (socketFd < 0) {
        std::cout << "Failed to create socket." << std::endl;
        return 1;
    }

    // Set up server address
    struct sockaddr_in serverAddress{};
    memset(&serverAddress, 0, sizeof(serverAddress));
    serverAddress.sin_family = AF_INET;
    serverAddress.sin_port = htons(serverPort);
    if (inet_pton(AF_INET, serverIP, &(serverAddress.sin_addr)) <= 0) {
        std::cout << "Invalid server IP address." << std::endl;
        return 1;
    }

    // Transmit audio data
    ssize_t numBytesSent = sendto(socketFd, audioBuffer, strlen(audioBuffer), 0,
                                  (struct sockaddr*) &serverAddress, sizeof(serverAddress));
    if (numBytesSent < 0) {
        std::cout << "Failed to send audio data." << std::endl;
        return 1;
    }

    // Close the socket
    close(socketFd);

    std::cout << "Audio transmission completed." << std::endl;

    return 0;
}
