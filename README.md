#!/usr/bin/env python3
"""
GIF (1x art resolution) -> crisp animated SVG sprite-sheet + still twin.
Creates lightweight SVGs suitable for responsive web design with crisp pixel art rendering.
"""

import argparse
import base64
import io
import sys
from PIL import Image, ImageSequence


def parse_args():
    parser = argparse.ArgumentParser(
        description="Convert animated GIFs to crisp animated SVG sprite sheets."
    )
    parser.add_argument("gif", help="Path to input animated GIF file")
    parser.add_argument(
        "out", help="Output stem path (e.g. assets/card)"
    )
    parser.add_argument(
        "--scale", type=int, default=2, help="Intrinsic SVG size multiple (default: 2)"
    )
    parser.add_argument(
        "--cols", type=int, default=8, help="Max columns in sprite sheet (default: 8)"
    )
    parser.add_argument(
        "--still", type=int, default=0, help="Frame index for still twin (default: 0)"
    )
    return parser.parse_args()


def image_to_base64_uri(img: Image.Image) -> str:
    """Converts a PIL Image object into a PNG base64 Data URI string."""
    buf = io.BytesIO()
    img.save(buf, "PNG", optimize=True)
    return "data:image/png;base64," + base64.b64encode(buf.getvalue()).decode()


def main():
    args = parse_args()

    # Load GIF and extract frames + frame durations
    try:
        im = Image.open(args.gif)
    except FileNotFoundError:
        sys.exit(f"Error: Input file '{args.gif}' not found.")

    frames, durs = [], []
    for frame in ImageSequence.Iterator(im):
        frames.append(frame.convert("RGBA"))
        # Fallback to 70ms if duration info is missing
        durs.append(frame.info.get("duration") or 70)

    n_frames = len(frames)
    if n_frames == 0:
        sys.exit("Error: No frames found in GIF.")

    w, h = frames[0].size
    cols = min(args.cols, n_frames)
    rows = -(-n_frames // cols)  # Ceiling division
    total_duration = sum(durs)

    # Build Sprite Sheet
    sheet = Image.new("RGBA", (cols * w, rows * h), (0, 0, 0, 0))
    for i, frame in enumerate(frames):
        sheet.paste(frame, ((i % cols) * w, (i // cols) * h))

    # SVG Metadata and CSS
    head = (
        f'<svg xmlns="http://www.w3.org/2000/svg" '
        f'width="{w * args.scale}" height="{h * args.scale}" '
        f'viewBox="0 0 {w} {h}" shape-rendering="crispEdges">'
    )
    css = "image{image-rendering:pixelated;image-rendering:crisp-edges}"

    # Generate CSS Keyframe steps
    elapsed_time, keyframe_stops = 0, []
    for i, duration in enumerate(durs):
        x, y = (i % cols) * w, (i // cols) * h
        percentage = (100 * elapsed_time) / total_duration
        keyframe_stops.append(f"{percentage:.4f}%{{transform:translate(-{x}px,-{y}px)}}")
        elapsed_time += duration

    # Build Animated SVG
    anim_svg = (
        f"{head}"
        f"<style>{css}.a{{animation:k {total_duration}ms step-end infinite}}"
        f"@keyframes k{{{''.join(keyframe_stops)}}}</style>"
        f'<defs><clipPath id="c"><rect width="{w}" height="{h}"/></clipPath></defs>'
        f'<g clip-path="url(#c)"><image class="a" width="{cols * w}" height="{rows * h}" '
        f'href="{image_to_base64_uri(sheet)}"/></g></svg>'
    )

    # Build Still SVG
    still_frame_idx = min(args.still, n_frames - 1)
    still_svg = (
        f"{head}"
        f"<style>{css}</style>"
        f'<image width="{w}" height="{h}" href="{image_to_base64_uri(frames[still_frame_idx])}"/></svg>'
    )

    # Output Files
    out_anim_path = f"{args.out}.svg"
    out_still_path = f"{args.out}-still.svg"

    with open(out_anim_path, "w", encoding="utf-8") as f:
        f.write(anim_svg)

    with open(out_still_path, "w", encoding="utf-8") as f:
        f.write(still_svg)

    # Summary Output
    print(f"{n_frames} frames, {w}x{h}px, {total_duration}ms loop")
    print(f" -> {out_anim_path} ({len(anim_svg) // 1024} KB)")
    print(f" -> {out_still_path} ({len(still_svg) // 1024} KB)\n")
    print("HTML Embed Snippet:")
    print(
        f'<picture>\n'
        f'  <source media="(prefers-reduced-motion: reduce)" srcset="{out_still_path}">\n'
        f'  <img src="{out_anim_path}" width="100%" alt="DESCRIBE THE SCENE">\n'
        f'</picture>'
    )


if __name__ == "__main__":
    main()
