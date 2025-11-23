# ProteinCHAOS
MD-inspired protein art generator in your browser

This software is a merge of scientific concepts and artistic interpretations. While efforts have been made to ensure accuracy, the generated visuals may not perfectly represent real-world protein dynamics. Think about it as a creative exploration rather than a precise scientific tool.

Inspired by long exposure photography, generative art, and molecular dynamics simulations, ProteinCHAOS brings proteins to life through motion and color.

<img width="896" height="761" alt="chaos_vc_3HDL_charcoal" src="https://github.com/user-attachments/assets/1d1d4977-0926-4bee-8549-adc62a19634b" />

<img width="670" height="578" alt="chaos_vc_Q9NYL2_ice" src="https://github.com/user-attachments/assets/7da9b085-4362-4c95-98b2-816137f68710" />


## ProteinCHAOS is a single-page web app that:

* Reads PDB (RCSB) and AlphaFold (AF-XXXX-F1-model_v6.pdb) structures.

* Builds a simplified coarse-grained model (Cα / P / glycan C1 atoms).

* Runs a stochastic dynamics kernel in a Web Worker.

* Projects motion into a 2D accumulation buffer to create long-exposure “protein trails”.

* Lets you tune physics and style interactively and export high-res stills or short videos.

* Think of it as “MD light + datashader + cover art generator” that happens entirely in the browser.

## What this is not
ProteinCHAOS is not a molecular dynamics package or structural modeling tool.

* It does not implement a physical force field or solvent.

* It does not preserve exact bond lengths, angles, or stereochemistry.

* It does not guarantee the biological correctness of glycan trees or loop motions.

It is an artistic engine inspired by structural biology and MD, designed for:

* Lab cover art and slides.

* Outreach and teaching visuals.

Interactive exploration of "how a protein might move" at a very high level.

<img width="2200" height="886" alt="chaos_vc_8PTU_fire" src="https://github.com/user-attachments/assets/1791419c-f3f2-4e26-9cec-a2c6e576bd36" />

<img width="5781" height="2160" alt="chaos_vc_8UT2_graphite" src="https://github.com/user-attachments/assets/73c2d5e8-c3a5-4a15-bc82-abb040993dee" />
<img width="2500" height="934" alt="chaos_vc_8UT2_magma" src="https://github.com/user-attachments/assets/4bdfc15d-1d68-4e7d-8f39-b91e0f1d4cfb" />

### Disclaimer
This project is not affiliated with or endorsed by RCSB or AlphaFold. All protein structures are sourced from publicly available databases. This software is intended for educational and artistic purposes only and was vibe coded using Gemini 3 Pro.