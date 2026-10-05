name: Build SV Video Editor
on: [push, workflow_dispatch]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          channel: 'stable'

      - name: Create SV Project
        run: |
          flutter create --org com.sv.editor sv_app
          cd sv_app
          flutter pub add video_player image_picker

      - name: Write SV Video Editor Code
        run: |
          cat > sv_app/lib/main.dart << 'EOF'
          import 'dart:io';
          import 'package:flutter/material.dart';
          import 'package:image_picker/image_picker.dart';
          import 'package:video_player/video_player.dart';

          void main() {
            runApp(const MaterialApp(
              title: 'SV',
              home: SVEditor(),
              debugShowCheckedModeBanner: false,
            ));
          }

          class SVEditor extends StatefulWidget {
            const SVEditor({super.key});

            @override
            State<SVEditor> createState() => _SVEditorState();
          }

          class _SVEditorState extends State<SVEditor> {
            File? _video;
            VideoPlayerController? _controller;
            final ImagePicker _picker = ImagePicker();
            RangeValues _range = const RangeValues(0, 10);
            double _totalDuration = 10;
            String _msg = "Select a video to edit in SV";

            Future<void> _pick() async {
              final XFile? file = await _picker.pickVideo(source: ImageSource.gallery);
              if (file != null) {
                _controller?.dispose();
                final vFile = File(file.path);
                final c = VideoPlayerController.file(vFile);
                await c.initialize();
                setState(() {
                  _video = vFile;
                  _controller = c;
                  _totalDuration = c.value.duration.inSeconds.toDouble();
                  _range = RangeValues(0, _totalDuration > 10 ? 10 : _totalDuration);
                  _msg = "Video loaded (${_totalDuration.toStringAsFixed(1)}s)";
                });
                c.play();
              }
            }

            @override
            void dispose() {
              _controller?.dispose();
              super.dispose();
            }

            @override
            Widget build(BuildContext context) {
              return Scaffold(
                backgroundColor: const Color(0xFF111111),
                appBar: AppBar(
                  backgroundColor: Colors.black,
                  title: const Text('SV Editor', style: TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
                  actions: [
                    if (_video != null)
                      TextButton(
                        onPressed: () {
                          ScaffoldMessenger.of(context).showSnackBar(
                            SnackBar(content: Text('Video trimmed from ${_range.start.toInt()}s to ${_range.end.toInt()}s')),
                          );
                        },
                        child: const Text('Export', style: TextStyle(color: Colors.blueAccent, fontWeight: FontWeight.bold)),
                      )
                  ],
                ),
                body: Column(
                  children: [
                    Expanded(
                      child: Center(
                        child: _controller != null && _controller!.value.isInitialized
                            ? AspectRatio(
                                aspectRatio: _controller!.value.aspectRatio,
                                child: VideoPlayer(_controller!),
                              )
                            : Text(_msg, style: const TextStyle(color: Colors.white54)),
                      ),
                    ),
                    if (_video != null) ...[
                      Padding(
                        padding: const EdgeInsets.symmetric(horizontal: 20),
                        child: RangeSlider(
                          values: _range,
                          min: 0,
                          max: _totalDuration > 0 ? _totalDuration : 1,
                          activeColor: Colors.blueAccent,
                          inactiveColor: Colors.white24,
                          onChanged: (v) {
                            setState(() => _range = v);
                            _controller?.seekTo(Duration(seconds: v.start.toInt()));
                          },
                        ),
                      ),
                      const SizedBox(height: 20),
                    ]
                  ],
                ),
                floatingActionButton: FloatingActionButton(
                  backgroundColor: Colors.blueAccent,
                  onPressed: _pick,
                  child: const Icon(Icons.add, color: Colors.white),
                ),
              );
            }
          }
          EOF

      - name: Build Secure Release APK
        run: |
          cd sv_app
          flutter build apk --release

      - name: Upload Artifact
        uses: actions/upload-artifact@v4
        with:
          name: SV-Release-APK
          path: sv_app/build/app/outputs/flutter-apk/app-release.apk
