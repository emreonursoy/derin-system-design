# Diagram: how to edit it live

- Source of truth: architecture.excalidraw. Open it in https://excalidraw.com (File > Open) or the VS Code Excalidraw extension.
- Image: export `architecture.png` after every change.

## Export to PNG
1. Open https://excalidraw.com, and choose menu > Open. Select `diagram/architecture.excalidraw`.
2. Check that the main diagram and the data panel beside it are both visible. Select nothing, so the whole canvas exports.
3. Choose menu > Export image. Turn **Background** on, and leave "Only selected" off. Use a scale of 2x if offered.
4. Choose **PNG**, and save it as `diagram/architecture.png`, replacing the old file.
5. Open the PNG, and check that all text is readable. The AI can also read it and list overlaps.

## Conventions
- Must show: components, request flows, data flows, technology choices.
- Conventions: solid arrow = request flow, dotted = data/async flow, dashed = optional. Each box names its technology.
- Live-change drill (practice before the interview): ask the AI to "add X / replace Y / move Z" by editing the .excalidraw file, then reopen it and check arrows still attach.