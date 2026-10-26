# Kai Ortadeveci: Research Portfolio

Cognitive Science, UC Berkeley (B.A. expected December 2027)
kaiorta@berkeley.edu · [linkedin.com/in/kaiortadeveci](https://www.linkedin.com/in/kaiortadeveci)

I work where neuroscience, machine learning and hardware meet. This page covers two research projects:

1. [MEA-CartPole](#1-mea-cartpole-neurotechberkeley): a closed-loop brain-computer interface that trains living neurons to play a video game.
2. [Reviving an unpublished visual cortex model](#2-reviving-an-unpublished-model-of-visual-cortex-usc-mel-lab): bringing a decade-old computational neuroscience PhD model back to life and extending it.

---

## 1. MEA-CartPole (Neurotech@Berkeley)

**Role:** Co-Lead · Feb 2026 to present
**Stack:** Python, C#, Gymnasium, Brian2, NumPy, SciPy, MCS MEA2100-Mini, Mcs.Usb SDK

> **Status:** Hardware brought up, pipeline audited, v2 encoding/decoding designed. No live-neuron sessions have run yet. Every result below comes from simulation or hardware checks, not from neurons learning the task. The project code lives in a private team repository.

### What it is

A closed-loop system that turns the state of a CartPole game into electrical stimulation of living neurons grown on a 60-electrode microelectrode array (MEA), then reads their spiking back as push-left or push-right moves.

### The research question

Cortical neurons on an array have improved at Pong (DishBrain, Kagan et al., 2022), and cortical organoids have shown goal-directed learning on CartPole (Robbins et al., 2026). That second result used a high-density array with 26,400 electrodes, which can find circuits that already exist and train them.

Our array has 60 electrodes at 200 µm spacing, about 11 times coarser, so each electrode hears a small cluster of cells rather than one neuron. We ask a different question: **can a dissociated culture learn an input-to-output mapping we impose on it**, forming a new pathway instead of strengthening one that already exists?

### How the loop works

| Step | What happens |
| --- | --- |
| 1. Game | CartPole reports the pole's angle and angular velocity. |
| 2. Encode | The state becomes a stimulation pattern: which electrode, at what pulse rate. |
| 3. Stimulate | Biphasic, charge-balanced pulses go into the dish. |
| 4. Record | All 60 electrodes are sampled at up to 50 kHz. |
| 5. Decode | Spikes on the left and right output electrodes pick the move. |
| 6. Feedback | A separate training signal, on its own generator, follows performance. |

Python runs the game, encoder and decoder. C# drives the MEA hardware through the vendor SDK. In v2, each game step gets its own 200 ms cycle.

### My work

**Brought up the hardware**
- Set up the MEA chip and headstage and confirmed raw data streaming from all 60 channels.
- Traced the full signal path (Python to C# to the SDK to the device) and logged where it broke: an interface file never wired into the main program, dummy data in place of electrode data, compile errors, and a hardcoded stimulation waveform.
- Showed from the hardware manual that the old amplitude encoding (120 to 180 µV) fit inside a single 305 µV DAC step, so amplitude carried no information.

**Audited the inherited pipeline by running it**

I ran the existing encoding and decoding code instead of only reading it, and documented 15 defects. The most important:
- **The game state never reached the neurons.** Radians were treated as degrees, so stimulation moved between 19.8 and 20.2 Hz instead of 10 to 30 Hz.
- **The loop rewarded itself.** Stimulation landed on the same electrodes used to decode the move, so whichever side won once would likely keep winning.
- **There was no real feedback.** The reward branch fired every step and the punishment branch could never run.
- **Decoding was off.** Filter cutoffs were wrong by about 20 times from a bad sampling-rate assumption, and the code counted samples instead of spikes.

**Redesigned encoding and decoding (v2)**

The core rule: input and output live in separate regions, so a move has to cross the network instead of echoing the stimulus.

```
         OUTPUT row:  L  L  L  |  R  R  R      (3 left vs 3 right decide the move)
                   .  .  .  .  .  .  .  .
                   .  .  .  .  .  .  .  .
                   .  .  .  .  .  .  .  .      network in between,
                   .  .  .  .  .  .  .  .      recorded throughout
                   .  .  .  .  .  .  .  .      (rows 1.4 mm apart)
                   .  .  .  .  .  .  .  .
          INPUT row:  1  2  3     4  5  6      (pole angle picks one electrode)
```

| Element | Old pipeline | v2 |
| --- | --- | --- |
| Pole angle | Pulse frequency, effectively unencoded | Which of 6 input electrodes (place code) |
| Angular velocity | Pulse amplitude, inside one DAC step | Signed pulse rate, 10 to 40 Hz |
| Stimulation site | 30 electrodes, the same ones used to decode | 1 electrode, chosen by game state |
| Decode | Channels 0 to 29 vs 30 to 59 by index | 3 vs 3 output electrodes by physical position, template-matched spikes |
| Feedback | None in practice | Structured or unstructured feedback from a rank rule, on its own generator |
| Timing | Real time, with stale data | One game step per 200 ms cycle |

**Tested every design choice in simulation first**

Before using hardware time, I simulated scripted CartPole policies and toy cultures in Gymnasium (300 to 1,000 episodes per test) and changed the design where the numbers said to:
- A 1° buffer at vertical would give a well-balancing controller the wrong side on **49%** of steps, hiding left and right exactly when play improves. I had proposed the buffer, then dropped it.
- Holding each decision for 200 ms in real time cut even a perfect controller from **500 steps to 10**, which is why v2 slows the game to one step per cycle.
- A toy culture that only spreads activity scored **42 steps** under normal mapping and **11** under mirrored mapping, so learning has to beat each dish's own baseline, not random play.

**Designed controls that separate learning from look-alikes**
- **Timing-shuffled feedback:** the same training signal, the same number of times, at random moments. I narrowed it to shuffle only feedback events, since shuffling the whole log would leave the culture playing blind and the control could never fail.
- **Mirrored mapping** in half the dishes, to rule out electrode geometry.
- **Mid-session rule change:** reverse the push direction. Real learning should drop, then recover.
- **Whole-dish firing logged every minute** to track homeostatic drift.

Several of these came from conversations I set up with Cortical Labs and a UC Berkeley neuroscience professor.

**Co-lead the team**
- Co-lead an 8-person team across 6 workstreams: encoding and decoding, hardware, closed-loop API and latency, controls, feedback, and network mapping.
- Built onboarding docs and pipeline diagrams for a codebase that had almost none, and a tracker of about 114 decisions, builds and tests that surfaced 5 conflicting proposals for the team to settle.

**Poster:** "Closed-Loop Control of the CartPole Task Using Neural Activity Recorded from Microelectrode Arrays." A. Sabouri, R. Wilson, K. Ortadeveci, E. Hou. Inter-UC neurotech conference, Spring 2026.

### Next steps

Map each channel to its physical electrode, run saline controls to set spike thresholds, measure loop latency against a 20 ms budget, wire v2 into the live loop, and run the first sessions with live cultures from our partner wet lab.

---

## 2. Reviving an unpublished model of visual cortex (USC, Mel Lab)

**Role:** Research Assistant, Laboratory for Neural Computation (Prof. Bartlett W. Mel) · Aug 2025 to present
**Stack:** MATLAB, Python, TensorFlow/Keras, Git, Claude Code

> **Note on disclosure:** This work is unpublished and is being prepared for publication with the original author and PI. I can't share results, figures, parameter values or code here. This section describes the project and my role at a high level only.

### What it is

A computational model, from a 2015 USC PhD dissertation, of how the brain learns to recognize visual features without any labels or teaching signal. Neurons in mid-level visual cortex (areas V2 and V4) respond to a huge variety of shapes and textures, and how that variety develops is an open question. The model proposes a simple two-stage mechanism:

1. **A threshold gate.** A simulated patch of primary visual cortex (V1) looks at small natural-image patches and passes on only the ones that excite it most strongly.
2. **Competitive learning.** A large population of model V4 cells learns from those patches. For each patch, only the most active cells adjust toward it. There are no labels and no backpropagation.

In its full form, the model works like a hybrid Kohonen self-organizing map with a threshold gate in the first layer.

### The challenge

The code had not run in about a decade, the scripts that set each experiment's parameters were lost, and the dissertation, the code and the code's comments disagreed with each other in many places. None of the three sources could be trusted alone.

### My role

- **Got it running again** on modern software and new natural-image datasets without changing its science, keeping the original files for comparison.
- **Traced the math of the entire model** from the dissertation into the code, and resolved the places where the paper and code disagreed using evidence from archived files.
- **Extended the hybrid learning rule** to natural images and tuned the model across a large parameter campaign.
- **Reproduced the dissertation's central claims** on new data.
- **Measured the model's computational cost** and made its first stage more efficient.
- **Built an AI-assisted research workflow** in Claude Code, with approval gates before any change, standardized reports, and every edit checked against the original.

Results will be shared once the work is published.
