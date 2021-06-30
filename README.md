# Assem-VC &mdash; Official PyTorch Implementation

![](./docs/images/overall.png)

**Assem-VC: Realistic Voice Conversion by Assembling Modern Speech Synthesis Techniques**<br>
Kang-wook Kim, Seung-won Park, Myun-chul Joe @ [MINDsLab Inc.](https://mindslab.ai), SNU

Paper: https://arxiv.org/abs/2104.00931 <br>
Audio Samples: https://mindslab-ai.github.io/assem-vc/ <br>

Abstract: *In this paper, we pose the current state-of-the-art voice conversion (VC) systems as two-encoder-one-decoder models. After comparing these models, we combine the best features and propose Assem-VC, a new state-of-the-art any-to-many non-parallel VC system. This paper also introduces the GTA finetuning in VC, which significantly improves the quality and the speaker similarity of the outputs. Assem-VC outperforms the previous state-of-the-art approaches in both the naturalness and the speaker similarity on the VCTK dataset. As an objective result, the degree of speaker disentanglement of features such as phonetic posteriorgrams (PPG) is also explored. Our investigation indicates that many-to-many VC results are no longer distinct from human speech and similar quality can be achieved with any-to-many models.*

## TODO List (2021.07.01)
- [ ] Enable GTA finetuning
- [ ] Upload loss curves
- [ ] Upload pre-trained weight

## Requirements

This repository was tested with following environment:

- Python 3.6.8
- [PyTorch](https://pytorch.org/) 1.4.0
- [PyTorch Lightning](https://github.com/PytorchLightning/pytorch-lightning) 1.0.3
- The requirements are highlighted in [requirements.txt](./requirements.txt).

## Clone our Repository
```bash
git clone --recursive https://github.com/mindslab-ai/assem-vc
cd assem-vc
```

## Datasets

### Preparing Data

- To reproduce the results from our paper, you need to download:
  - LibriTTS train-clean-100 split [tar.gz link](http://www.openslr.org/resources/60/train-clean-100.tar.gz)
  - [VCTK dataset (Version 0.80)](https://datashare.ed.ac.uk/handle/10283/2651)
- Unzip each files.
- Resample them into 22.05kHz using `datasets/resample.py`.

### Preparing Metadata

Following a format from [NVIDIA/tacotron2](https://github.com/NVIDIA/tacotron2), the metadata should be formatted like:
```
path_to_wav|transcription|speaker_id
path_to_wav|transcription|speaker_id
...
```

When you want to learn and inference using phoneme, the transcription should have only unstressed [ARPABET](https://en.wikipedia.org/wiki/ARPABET).

Metadata containing ARPABET for LibriTTS train-clean-100 split and VCTK corpus are already prepared at `datasets/metadata`.
If you wish to use custom data, you need to make the metadata as shown above.

When converting transcription of metadata into ARPABET, you can use `datasets/g2p.py`.

```bash
python datasets/g2p.py -i <input_metadata_filename_with_graphemes> -o <output_filename>
```

### Preparing Configuration Files

Training our VC system is consisted of two steps: (1) training Cotatron, (2) training VC decoder on top of Cotatron.

There are three `yaml` files in the `config` folder, which are configuration template for each model.
They **must** be edited to match your training requirements (dataset, metadata, etc.).

```bash
cp config/global/default.yaml config/global/config.yaml
cp config/cota/default.yaml config/cota/config.yaml
cp config/vc/default.yaml config/vc/config.yaml
```

Here, all files with name other than `default.yaml` will be ignored from git (see `.gitignore`).

- `config/global`: Global configs that are both used for training Cotatron & VC decoder.
  - Fill in the blanks of: `speakers`, `train_dir`, `train_meta`, `val_dir`, `val_meta`.
  - Example of speaker id list is shown in `datasets/metadata/libritts_vctk_speaker_list.txt`.
  - When replicating the two-stage training process from our paper (training with LibriTTS and then LibriTTS+VCTK), please put both list of speaker ids from LibriTTS and VCTK at global config.
- `config/cota`: Configs for training Cotatron.
  - You may want to change: `batch_size` for GPUs other than 32GB V100, or change `chkpt_dir` to save checkpoints in other disk.
- `config/vc`: Configs for training VC decoder.
  - Fill in the blank of: `cotatron_path`. 

### Extracting Pitch Range of Speakers

Before you train VC decoder, you should extract pitch range of each speaker:

```bash
python preprocess.py -c <path_to_global_config_yaml>
```
Result will be saved at `f0s.txt`.

## Training

### 1. Training Cotatron

To train the Cotatron, run this command:

```bash
python cotatron_trainer.py -c <path_to_global_config_yaml> <path_to_cotatron_config_yaml> \
                           -g <gpus> -n <run_name>
```

Here are some example commands that might help you understand the arguments:

```bash
# train from scratch with name "my_runname"
python cotatron_trainer.py -c config/global/config.yaml config/cota/config.yaml \
                           -g 0 -n my_runname
```

Optionally, you can resume the training from previously saved checkpoint by adding `-p <checkpoint_path>` argument.

### 2. Training VC decoder

After the Cotatron is sufficiently trained (i.e., producing stable alignment + converged loss),
the VC decoder can be trained on top of it.

```bash
python synthesizer_trainer.py -c <path_to_global_config_yaml> <path_to_vc_config_yaml> \
                              -g <gpus> -n <run_name>
```

The optional checkpoint argument is also available for VC decoder.

### 3. GTA finetuning HiFi-GAN

TBD

### Monitoring via Tensorboard

The progress of training with loss values and validation output can be monitored with tensorboard.
By default, the logs will be stored at `logs/cota` or `logs/vc`, which can be modified by editing `log.log_dir` parameter at config yaml file.

```bash
tensorboard --log_dir logs/cota --bind_all # Cotatron - Scalars, Images, Hparams, Projector will be shown.
tensorboard --log_dir logs/vc --bind_all # VC decoder - Scalars, Images, Hparams will be shown.
```

## Inference

After the VC decoder and HiFi-GAN are trained, you can use an arbitrary speaker's speech as the source.
You can convert it to speaker contained in trainset: which is any-to-many voice conversion.
1. Add your source audio(.wav) in `datasets/inference_source`
2. Add following lines at `datasets/inference_source/metadata_origin.txt`
    ```
    your_audio.wav|transcription|speaker_id
    ```
    Note that speaker_id has no effect whether or not it is in the training set.
3. Convert `datasets/inference_source/metadata_origin.txt` into ARPABET.
    ```bash
    python3 datasets/g2p.py -i datasets/inference_source/metadata_origin.txt \
                            -o datasets/inference_source/metadata_g2p.txt
    ```
4. Run [inference.ipynb](./inference.ipynb)

## Results

![](./docs/images/results.png)

*Disclaimer: We used an open-source g2p system in this repository, which is different from the proprietary g2p mentioned in the paper.
Hence, the quality of the result may differ from the paper.*

## Implementation details

Here are some noteworthy details of implementation, which could not be included in our paper due to the lack of space:

- Guided attention loss

We applied guided attention loss proposed in [DC-TTS](https://arxiv.org/abs/1710.08969).
It helped Cotatron's alignment learning stable and faster convergence.
See [utils/alignment_loss.py](./utils/alignment_loss.py).

## License

BSD 3-Clause License.

## Citation & Contact

```bibtex
@article{kim2021assem,
  title={Assem-VC: Realistic Voice Conversion by Assembling Modern Speech Synthesis Techniques},
  author={Kim, Kang-wook and Park, Seung-won and Joe, Myun-chul},
  journal={arXiv preprint arXiv:2104.00931},
  year={2021}
}
```

If you have a question or any kind of inquiries, please contact Kang-wook Kim at [kwkim@mindslab.ai](mailto:kwkim@mindslab.ai)


## Repository structure

TBD

## References

This implementation uses code from following repositories:
- [Keith Ito's Tacotron implementation](https://github.com/keithito/tacotron/)
- [NVIDIA's Tacotron2 implementation](https://github.com/NVIDIA/tacotron2)
- [Official Mellotron implementation](https://github.com/NVIDIA/mellotron)
- [Official HiFi-GAN implementation](https://github.com/jik876/hifi-gan)
- [Official Cotatron implementation](https://github.com/mindslab-ai/cotatron)
- [Kyubyong's g2pE implementation](https://github.com/Kyubyong/g2p)
- [Tomiinek's Multilngual TTS implementation](https://github.com/Tomiinek/Multilingual_Text_to_Speech)

This README was inspired by:
- [Tips for Publishing Research Code](https://github.com/paperswithcode/releasing-research-code)

The audio samples on our [webpage](https://mindslab-ai.github.io/cotatron/) are partially derived from:
- [LibriTTS](https://arxiv.org/abs/1904.02882): Dataset for multispeaker TTS, derived from LibriSpeech.
- [VCTK](https://homepages.inf.ed.ac.uk/jyamagis/page3/page58/page58.html): 46 hours of English speech from 108 speakers.
- [KSS](https://www.kaggle.com/bryanpark/korean-single-speaker-speech-dataset): Korean Single Speaker Speech Dataset.
