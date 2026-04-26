name: Snake 🐍

on:
  schedule:
    - cron: "0 0 * * *"
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  generate:
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Generate contribution snake (light + dark)
        uses: Platane/snk/svg-only@v3
        with:
          github_user_name: ${{ github.repository_owner }}
          outputs: |
            dist/github-snake.svg?color_snake=6B4423&color_dots[0]=161b22&color_dots[1]=2d3a1e&color_dots[2]=4a6128&color_dots[3]=6b8e3a&color_dots[4]=8fad52&color_empty=0d1117
            dist/github-snake-dark.svg?color_snake=8B6914&color_dots[0]=161b22&color_dots[1]=3b2f0e&color_dots[2]=5c4a1a&color_dots[3]=8c6e24&color_dots[4]=c4a040&color_empty=0d1117

      - name: Inject score counter into snake SVG
        run: |
          python3 - <<'EOF'
          import re, os

          for fname in ["dist/github-snake.svg", "dist/github-snake-dark.svg"]:
              if not os.path.exists(fname):
                  continue
              with open(fname, "r") as f:
                  svg = f.read()

              # Count food dots (contribution cells that are non-empty)
              food_count = len(re.findall(r'class="[^"]*\bcontrib\b[^"]*"', svg))
              if food_count == 0:
                  # Fallback: count any colored rect/circle that is not empty
                  food_count = len(re.findall(r'fill="(?!#161b22|#0d1117)[^"]{4,7}"', svg))

              score_label = f"SCORE  {food_count:04d}"

              # Find SVG viewBox width to position score top-right
              vb = re.search(r'viewBox="[^"]*"', svg)
              svg_w = 800
              if vb:
                  parts = vb.group().replace('viewBox="', '').replace('"', '').split()
                  if len(parts) == 4:
                      svg_w = int(float(parts[2]))

              score_svg = f'''
            <g font-family="monospace" font-size="12" fill="#8B6914">
              <rect x="{svg_w - 140}" y="8" width="132" height="22" rx="4"
                    fill="#0d1117" stroke="#3b2f0e" stroke-width="1"/>
              <text x="{svg_w - 74}" y="23" text-anchor="middle"
                    letter-spacing="2" font-weight="bold">{score_label}</text>
            </g>'''

              # Inject before closing </svg>
              svg = svg.replace("</svg>", score_svg + "\n</svg>")

              with open(fname, "w") as f:
                  f.write(svg)
              print(f"Patched {fname} — score: {food_count}")
          EOF

      - name: Push to output branch
        uses: crazy-max/ghaction-github-pages@v3.1.0
        with:
          target_branch: output
          build_dir: dist
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
