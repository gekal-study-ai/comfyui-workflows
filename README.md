# comfyui-workflows

ComfyUI で使っているワークフロー（`.json`）の置き場です。画像生成と動画生成のテンプレートを、手元の環境で動く設定に調整したものを保存しています。

各ワークフローは ComfyUI の **サブグラフ**（[Subgraph](https://docs.comfy.org/interface/features/subgraph)）機能で本体処理をまとめてあり、トップレベルにはプロンプト・入力画像・解像度・保存先といった「触る部分」だけが出ています。

## 使い方

1. ComfyUI を最新版に更新する（[更新手順](https://docs.comfy.org/installation/update_comfyui)）
2. 下記の一覧から使いたい JSON を選び、ComfyUI の画面にドラッグ＆ドロップするか、メニューの `Workflow > Open` から読み込む
3. 各ワークフローの節に書いてあるモデルを `ComfyUI/models/` 配下の所定の場所に配置する
4. 画像入力が必要なものは `Load Image` に自分の画像を差し替える（リポジトリに入っている JSON には作業時の画像名が残っています）
5. `Run` で実行。出力は `ComfyUI/output/` 配下に保存されます

> モデルが未配置だとノード上のファイル選択が赤くなります。その場合はファイル名と保存先を見直してください。

## ワークフロー一覧

| ファイル | 種別 | モデル | 出力先 |
|---|---|---|---|
| [`image_z_image.json`](image_z_image.json) | Text to Image | Z-Image (Base) | `output/Z_image_base_*.png` |
| [`video_wan2_2_14B_i2v.json`](video_wan2_2_14B_i2v.json) | Image to Video | Wan2.2 I2V 14B | `output/video/Wan2.2_i2v_*` |
| [`video_minimax_h3_i2v.json`](video_minimax_h3_i2v.json) | Image to Video（音声付き） | MiniMax H3 | `output/video/MiniMax_H3_*` |
| [`video_ltx2_3_t2v.json`](video_ltx2_3_t2v.json) | Text to Video（音声付き） | LTX-2.3 22B | `output/video/LTX_2.3_t2v_*` |

---

## image_z_image.json — Text to Image (Z-Image)

テキストから静止画を生成します。サブグラフ名は `Text to Image(Z-Image-Base Int8)`。

**トップレベルで触るノード**

- サブグラフ入力のプロンプト（ポジティブ / ネガティブ）
- `Resolution Selector`: アスペクト比・メガピクセル数・丸め単位（初期値 `1:1 (Square)` / `1` / `8`）
- `Save Image Advanced`: プレフィックス `Z_image_base`、`png` / 8-bit / sRGB

**サブグラフ内の主な設定**

| 項目 | 値 |
|---|---|
| Diffusion Model | `z_image_bf16.safetensors` |
| Text Encoder | `qwen_3_4b.safetensors`（type: `lumina2`） |
| VAE | `ae.safetensors` |
| Latent | `EmptySD3LatentImage` 1024 x 1024 |
| Sampler | `KSampler` / steps 25 / cfg 4 / `res_multistep` + `simple` |
| Model Sampling | `ModelSamplingAuraFlow` shift 3 |

ワークフロー内のメモにある推奨値は **steps 30〜50 / cfg 3〜5** です。初期値（25 / 4）はその下限寄りなので、品質を上げたいときは steps を増やします。

**必要なモデル**

```
ComfyUI/
└── models/
    ├── diffusion_models/z_image_bf16.safetensors
    ├── text_encoders/qwen_3_4b.safetensors
    └── vae/ae.safetensors
```

- [z_image_bf16.safetensors](https://huggingface.co/Comfy-Org/z_image/resolve/main/split_files/diffusion_models/z_image_bf16.safetensors)
- [qwen_3_4b.safetensors](https://huggingface.co/Comfy-Org/z_image_turbo/resolve/main/split_files/text_encoders/qwen_3_4b.safetensors)
- [ae.safetensors](https://huggingface.co/Comfy-Org/z_image_turbo/resolve/main/split_files/vae/ae.safetensors)

---

## video_wan2_2_14B_i2v.json — Image to Video (Wan2.2 14B)

1枚の画像から動画を生成します。サブグラフ名は `Image to Video (Wan2.2)`。high noise / low noise の2モデルを前半・後半で切り替える2段サンプリング構成です。

**トップレベルで触るノード**

- `Load Image`: 元画像
- サブグラフ入力: プロンプト / 幅・高さ（初期値 640 x 640）/ 尺（5秒）/ シード / 使用するモデル名
- `Save Video`: プレフィックス `video/Wan2.2_i2v`

**サブグラフ内の主な設定**

| 項目 | 通常 | 4steps LoRA 有効時 |
|---|---|---|
| Steps | 20 | 4 |
| Split Step（high→low の切替点） | 10 | 2 |
| CFG | 3.5 | 1 |
| Sampler | `euler` + `simple` | 同じ |

- `Enable 4steps LoRA?`（`PrimitiveBoolean`）を `true` にすると、Lightx2v の 4steps LoRA を high / low 両方に適用し、上表の右列の設定に切り替わります。初期値は `false`（LoRA なし）。
- フレーム数は `floor(FPS * Duration + 1)`。初期値は 16fps x 5秒 = 81 フレームで、`WanImageToVideo` の設定と一致します。
- ネガティブプロンプトは Wan の公式テンプレート由来の中国語のものが入っています。
- 出力は `CreateVideo` で 16fps。

**参考: RTX 4090D 24GB での実測**（ワークフロー内メモより、640 x 640）

| 構成 | VRAM | 1回目 | 2回目 |
|---|---|---|---|
| fp8_scaled | 84% | 約 536秒 | 約 513秒 |
| fp8_scaled + 4steps LoRA | 83% | 約 97秒 | 約 71秒 |

**必要なモデル**

```
ComfyUI/
└── models/
    ├── diffusion_models/
    │   ├── wan2.2_i2v_high_noise_14B_fp16.safetensors
    │   └── wan2.2_i2v_low_noise_14B_fp16.safetensors
    ├── loras/
    │   ├── wan2.2_i2v_lightx2v_4steps_lora_v1_high_noise.safetensors
    │   └── wan2.2_i2v_lightx2v_4steps_lora_v1_low_noise.safetensors
    ├── text_encoders/umt5_xxl_fp8_e4m3fn_scaled.safetensors
    └── vae/wan_2.1_vae.safetensors
```

配布元は [Comfy-Org/Wan_2.2_ComfyUI_Repackaged](https://huggingface.co/Comfy-Org/Wan_2.2_ComfyUI_Repackaged)（Text Encoder は [Wan_2.1_ComfyUI_repackaged](https://huggingface.co/Comfy-Org/Wan_2.1_ComfyUI_repackaged)）。公式チュートリアルは [Wan2.2](https://docs.comfy.org/tutorials/video/wan/wan2_2)。

---

## video_minimax_h3_i2v.json — Image to Video (MiniMax H3)

[MiniMax H3](https://www.minimax.io/blog/minimax-h3) で、**ステレオ音声付き**の動画を生成します。サブグラフ名は `Image to Video (MiniMax H3)`。映像と音声（セリフ・効果音・音楽）を1回の推論で同時に生成するのが特徴です。

`MiniMaxH3ImageToVideo` ノードは入力の繋ぎ方で挙動が変わります。

- 画像を繋がなければ **t2va**（テキスト → 映像+音声）
- `first_frame` / `last_frame` を繋げば **fl2va**（最初/最後のフレーム指定の i2v）。間の動きをモデルが補完します

**トップレベルで触るノード**

- `Load Image` → `Image Scale to Total Pixels`（0.9MP / 32 の倍数）→ `Get Image Size` で、入力画像から幅・高さを決めています
- `Resolution Selector`: 初期値 `1:1 (Square)` / 0.4MP / 32 の倍数
- サブグラフ入力: プロンプト、幅・高さ、尺
- `Save Video`: プレフィックス `video/MiniMax_H3`

**解像度の目安**（ワークフロー内メモより。H3 のネイティブは短辺 768px、上限 768 x 1344、32 の倍数に丸め）

| メガピクセル | 16:9 の出力 |
|---|---|
| 0.2 | 608 x 352 |
| 0.4 | 864 x 480 |
| 0.6 | 1056 x 608 |
| 0.8 | 1216 x 672 |
| 0.98 | 1344 x 768（公式 768p） |

**サブグラフ内の主な設定**

| 項目 | 値 |
|---|---|
| Diffusion Model | `minimax_h3_fl2va_pruned_int8_convrot.safetensors` |
| Text Encoder | `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors`（type: `minimax`） |
| VAE（映像 / 音声） | `minimax_h3_video_vae_int8_convrot.safetensors` / `minimax_h3_audio_vae_fp32.safetensors` |
| Steps | 20（Lightning LoRA 有効時は 6） |
| Sampler | `res_multistep` + `simple`（denoise 1） |
| 尺 | 2秒（`Float (duration)`） |
| 出力 | `CreateVideo` 24fps + 音声 |

- `Boolean (Enable Lightning LoRA)` を `true` にすると `minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors` を適用し、steps が 6 に切り替わります。初期値は `false`。
- フレーム数は `max(5, round(duration * 24))` を **17k+5** のグリッド（H3 の 17フレーム/ブロック）に切り上げて算出しています。尺を変えるとここが自動追従します。
- プロンプトは「映像の描写 + タイムライン + 音声（セリフ・SFX・BGM）」を1ブロックにまとめて書く形です。初期値にタイムライン付きのサンプルが入っているので、書き方の参考にしてください。

**必要なモデル**

```
ComfyUI/
└── models/
    ├── diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors
    ├── text_encoders/qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors
    ├── loras/
    │   ├── minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors
    │   └── minimax_h3_fl2v_turbo_4step_v1.0_768p_comfyui_bf16.safetensors
    ├── vae/
    │   ├── minimax_h3_video_vae_int8_convrot.safetensors
    │   └── minimax_h3_audio_vae_fp32.safetensors
    └── embeddings/minimaxh3_*.safetensors
```

- 本体・VAE・Text Encoder: [Comfy-Org/MiniMax-H3](https://huggingface.co/Comfy-Org/MiniMax-H3)
- 8steps LoRA: [lightx2v/Minimax-h3-Turbo](https://huggingface.co/lightx2v/Minimax-h3-Turbo)
- スタイル embedding（`minimaxh3_art_is_explosion` など10種）: [embeddings](https://huggingface.co/Comfy-Org/MiniMax-H3/tree/main/embeddings)

---

## video_ltx2_3_t2v.json — Text to Video (LTX-2.3)

[LTX-2.3](https://huggingface.co/Lightricks/LTX-2.3/) 22B で、**音声付き**の動画をテキストから生成します。サブグラフ名は `Text to Video (LTX-2.3)`。4つのワークフローの中で最も構成が大きく、以下を1本に繋いでいます。

1. `TextGenerateLTX2Prompt` によるプロンプト自動拡張（Gemma 3 12B + abliterated LoRA）
2. 低解像度での1段目サンプリング（映像 + 音声のラテントを結合したまま処理）
3. `LTXVLatentUpsampler` + 空間アップスケーラでの2段目サンプリング
4. `VAEDecodeTiled` で映像、`LTXVAudioVAEDecode` で音声をデコードし `CreateVideo` で合成

**トップレベルで触るノード**

- サブグラフ入力のプロンプト
- `Save Video`: プレフィックス `video/LTX_2.3_t2v`

**サブグラフ内の主な設定**

| 項目 | 値 |
|---|---|
| Checkpoint | `ltx-2.3-22b-dev.safetensors` |
| Text Encoder | `gemma_3_12B_it_fp4_mixed.safetensors` |
| 距離 LoRA | `ltx_2.3_22b_distilled_1.1_lora_dynamic_fro09_avg_rank_111_bf16.safetensors`（strength 0.5） |
| プロンプト拡張 LoRA | `gemma-3-12b-it-abliterated_lora_rank64_bf16.safetensors` |
| アップスケーラ | `ltx-2.3-spatial-upscaler-x2-1.1.safetensors` |
| 出力サイズ | 1280 x 720（`Width` / `Height`） |
| 尺 / FPS | 5秒 / 25fps（フレーム数 = `a * b + 1` = 126） |
| 1段目 latent | `EmptyLTXVLatentVideo` 768 x 512 / 97 フレーム |
| Sampler | `euler` + `ManualSigmas`（1段目9点 / 2段目4点）、CFG 1 |

- `Switch to Text to Video?` が `true` の間は t2v として動作します（`false` にすると画像入力側の経路に切り替わります）。
- `Boolean (Enable Prompt Enhance)` が `true` のとき `TextGenerateLTX2Prompt` がプロンプトを書き換えます。結果は `PreviewAny` で確認できます。
- ネガティブプロンプトには `pc game, console game, video game, cartoon, childish, ugly` が入っています。
- `CreateVideo` は 24fps 出力です（`Frame Rate` の 25 とは別に指定されています）。

**プロンプトのコツ**（ワークフロー内メモより）

1. **動作**: 時間の流れに沿って何が起きるかを書く
2. **見た目**: 画面に出したい視覚的要素をすべて書く
3. **音**: シーンに必要な効果音やセリフを書く

**必要なモデル**

```
ComfyUI/
└── models/
    ├── checkpoints/ltx-2.3-22b-dev.safetensors
    ├── loras/
    │   ├── ltx_2.3_22b_distilled_1.1_lora_dynamic_fro09_avg_rank_111_bf16.safetensors
    │   └── gemma-3-12b-it-abliterated_lora_rank64_bf16.safetensors
    ├── text_encoders/gemma_3_12B_it_fp4_mixed.safetensors
    └── latent_upscale_models/ltx-2.3-spatial-upscaler-x2-1.1.safetensors
```

- Checkpoint / アップスケーラ: [Lightricks/LTX-2.3](https://huggingface.co/Lightricks/LTX-2.3/)
- LoRA: [Comfy-Org/ltx-2.3](https://huggingface.co/Comfy-Org/ltx-2.3)、[Comfy-Org/ltx-2](https://huggingface.co/Comfy-Org/ltx-2)
- カスタムノード: [ComfyUI-LTXVideo](https://github.com/Lightricks/ComfyUI-LTXVideo)（不具合報告もこちら）

---

## 不具合の報告先

まず ComfyUI を更新し、必要なモデルが揃っているか確認してください。Desktop / Cloud 版は stable リリース追従のため、nightly でしか動かないモデルもあります。

- 実行できない / エラーが出る: [ComfyUI/issues](https://github.com/comfyanonymous/ComfyUI/issues)
- UI・フロントエンドの問題: [ComfyUI_frontend/issues](https://github.com/Comfy-Org/ComfyUI_frontend/issues)
- ワークフローテンプレートの問題: [workflow_templates/issues](https://github.com/Comfy-Org/workflow_templates/issues)
