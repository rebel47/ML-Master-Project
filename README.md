# Historical OCR Correction with Vision-Language Models

An exploratory research project investigating whether a vision-language model (VLM) can correct recognition errors in historical German documents by using both a cropped text-line image and the transcription produced by a conventional OCR engine.

**Current status:** Initial experiments completed on a small development dataset. Results are preliminary and are **not** evidence of generalization to other documents.

## Research questions

1. Can a VLM improve historical text recognition when it receives the original line image together with Tesseract's OCR output?
2. Can OCR confidence be used to select only the lines that need correction, reducing VLM calls while preserving accuracy?
3. When does VLM correction introduce errors, such as changing historical spelling or punctuation?
4. **Planned:** How much does the image contribute compared with text-only correction or image-only transcription?

## Pipeline

```text
Historical document page + PAGE-XML ground truth
                     |
               Crop text lines
                     |
              Tesseract OCR
             (German, PSM 7)
                     |
          OCR text + confidence
                     |
          +----------+----------+
          |                     |
    No correction       Qwen3-VL correction
          |             (line image + OCR text)
          |                     |
          +----------+----------+
                     |
          Compare against ground truth
                  CER / WER
```

For **selective correction**, the Qwen3-VL output is used only when the Tesseract confidence is below a chosen threshold; otherwise, the original Tesseract transcription is retained.

## Dataset

The initial experiment uses four pages from the historical German document `bebel_frau_1879` in the **OCR-D Ground Truth** collection.

| Property | Value |
| --- | --- |
| Document | `bebel_frau_1879` |
| Pages | 4 |
| Cropped, annotated text lines | 164 |
| Ground truth | PAGE-XML line transcriptions |
| Image input | Cropped text-line images |
| Language | German (historical orthography) |

The four pages are **from the same document**, so this is a limited development dataset rather than an independent, multi-document benchmark. Check the original dataset's terms and attribution requirements before redistributing source images or annotations.

## Models and evaluation

- **OCR baseline:** Tesseract 5.5.3, German language data (`deu`), page segmentation mode 7 (single text line).
- **Correction model:** `Qwen3-VL-8B-Instruct`, run with 4-bit NF4 quantization on a Google Colab Tesla T4 GPU.
- **Input to correction model:** Cropped line image and Tesseract transcription.
- **Output:** One corrected line of text, with instructions to preserve historical spelling and punctuation.
- **Metrics:** Corpus-level character error rate (**CER**) and word error rate (**WER**) relative to PAGE-XML ground truth. Lower is better.

### Full correction results

| Method | CER ↓ | WER ↓ | Lines sent to VLM |
| --- | ---: | ---: | ---: |
| Tesseract only | 1.34% | 8.25% | 0 |
| Tesseract + Qwen3-VL | **0.30%** | **2.40%** | 164 |

Full VLM correction achieved an approximately **77.5% relative reduction in CER** compared with Tesseract alone on this dataset. These percentage values are rounded for display; evaluation code computes metrics directly from predictions.

Across 164 lines, VLM correction resulted in:

- **77** lines with lower CER (improved).
- **79** lines with unchanged CER (which does not necessarily mean identical output text).
- **8** lines with higher CER (worsened).

Observed regressions included changes to historical spelling and spacing. Such modifications may be acceptable in modernized text but are incorrect for faithful diplomatic transcription.

### Selective correction results

A line is sent to the VLM when `ocr_confidence < threshold`.

| Confidence threshold | VLM calls | Selected lines | CER ↓ | WER ↓ |
| --- | ---: | ---: | ---: | ---: |
| 70 | 4 | 2.44% | 1.2532% | 7.9840% |
| 80 | 14 | 8.54% | 1.0492% | 7.0526% |
| 85 | 24 | 14.63% | 0.9909% | 6.6534% |
| 90 | 59 | 35.98% | 0.7189% | 5.1231% |
| 95 | 141 | 85.98% | 0.3303% | 2.6613% |
| Full correction | 164 | 100% | ~0.30% | ~2.40% |

At a threshold of **90**, selective correction reduced CER to approximately **0.72%** using **59 instead of 164 VLM calls** (approximately **64% fewer calls**). At **95**, it approached full-correction accuracy but required 141 calls. Call counts are a measure of usage, **not a direct measurement of runtime or cost**.

### VLM regressions under confidence gating

| Threshold | Regressions applied | Regressions avoided |
| --- | ---: | ---: |
| 70 | 0 | 8 |
| 80 | 0 | 8 |
| 85 | 1 | 7 |
| 90 | 3 | 5 |
| 95 | 7 | 1 |
| Full correction | 8 | 0 |

Conservative thresholds can avoid harmful corrections, but they also miss beneficial corrections. These figures count lines with higher CER after correction; they do not quantify the severity of each regression.

## Notebooks

Run the notebooks in sequence:

| Notebook | Purpose |
| --- | --- |
| `01_dataset_exploration.ipynb` | Inspect PAGE-XML annotations, extract line images, and create line metadata. |
| `02_ocr_baseline.ipynb` | Run Tesseract and evaluate baseline CER, WER, and OCR confidence. |
| `03_vlm_correction.ipynb` | Run Qwen3-VL correction and save line-level predictions. |
| `04_evaluation.ipynb` | Evaluate confidence thresholds, correction regressions, and accuracy–usage plots. |

Notebook filenames and dataset paths may need adjustment to match your local or Colab setup. The VLM notebook requires a supported GPU environment, the corresponding model dependencies, the baseline CSV, and the cropped line images. Reproducing results also requires the same model revision, prompt, generation settings, preprocessing, and dataset split.

## Limitations

- Only **164 lines from four pages of a single document** have been evaluated.
- Confidence thresholds have been explored on the same data; the results should not be treated as unbiased final test performance.
- The model can introduce historical-spelling, punctuation, and spacing changes.
- **Image-only and text-only ablations are not yet complete.** The current results demonstrate improvement from the combined pipeline, but do not isolate the contribution of visual grounding.
- VLM call counts have been compared, but end-to-end runtime and memory/cost comparisons are not yet reported.

## Next steps

1. Compare **image-only transcription**, **text-only OCR correction**, and **image + OCR correction** under matched settings.
2. Evaluate across additional documents, ideally with differing print styles, scan quality, and historical orthography.
3. Analyze correction gains and regressions by OCR confidence and error type.
4. Report inference-time trade-offs alongside CER/WER and call counts.

## Data and attribution

The source document is from the **OCR-D Ground Truth** collection (`bebel_frau_1879`). Please refer to the original collection for licensing, attribution, and redistribution terms. Source images and ground-truth files should only be committed if their licenses permit it.

## Project status

Research prototype / exploratory evaluation. Further datasets, ablation experiments, and the final project scope are to be discussed with the academic supervisor.
