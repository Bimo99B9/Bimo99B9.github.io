---
layout: single
title: "Jailbreaking Vision-Language Models with a Multimodal Attack"
excerpt: "We developed a joint adversarial attack that combines image and text perturbations to bypass the safety filters of advanced Vision-Language Models (VLMs) like LLaVA and Gemma."
date: 2025-06-09
classes: wide
header:
  teaser: /assets/images/multimodal-attacks/attack-example.png
  teaser_home_page: true
  icon: /assets/images/ai-security-icon.png
categories:
  - ai-security
  - machine-learning
tags:  
  - adversarial-attacks
  - vlm
  - llava
  - gemma
  - pgd
  - gcg
  - multimodal
  - jailbreaking
---

Vision-Language Models (VLMs) like LLaVA and Google's Gemma are incredibly powerful, demonstrating a deep understanding of both images and text. You can show them a picture and ask a question, and they'll often give you a stunningly accurate answer. However, this power comes with a hidden risk: they are vulnerable to adversarial attacks. An attacker could use carefully crafted inputs to trick these models into generating malicious or unsafe content, like instructions for building a bomb.

Most existing attacks target only one modality at a time—either by subtly altering an image or by manipulating the text prompt. In this project, we asked a simple question: what happens if we attack both at the same time?

### Our Unified Attack: Combining Image and Text Perturbations

We developed a **unified multimodal adversarial attack** that simultaneously targets both the vision and language inputs of a VLM. Our method combines two powerful techniques:

* **Projected Gradient Descent (PGD)**: A classic image attack that makes tiny, often human-imperceptible changes to an image's pixels to fool a model.
* **Greedy Coordinate Gradient (GCG)**: A state-of-the-art text attack that appends an optimized, often nonsensical-looking string of characters to a prompt to "jailbreak" a language model.

By integrating PGD and GCG into a single, cohesive framework, we can create adversarial image-text pairs that are far more effective than those from separate, unimodal attacks.

The difference is stark. In the example below, the LLaVA model correctly refuses to answer the harmful prompt when given a normal image. But when we apply our joint attack—making subtle changes to the image and adding an adversarial suffix to the prompt—the model's safety filter is completely bypassed, and it provides a detailed, harmful response.

![](/assets/images/multimodal-attacks/attack-example.png)
*A joint PGD+GCG attack tricks the LLaVA-RC model into providing illicit instructions, bypassing its safety alignment.*

---

### How the Joint Attack Works

The joint attack uses **simultaneous optimization**. Instead of attacking the image and text separately, the algorithm finds the best combination of perturbations in a single, efficient process. At a high level, here’s how it works:

1.  **Gradient Calculation**: For a given image and prompt, we perform a single backward pass to calculate the gradients for both the image pixels and the text token embeddings. This tells us the most effective direction to "push" both modalities to induce a jailbreak.
2.  **Image Update (PGD)**: We use the image gradient to apply a small, constrained perturbation to the input image, creating an adversarial version.
3.  **Text Candidate Generation (GCG)**: We use the text gradients to identify a set of top candidate tokens that are most likely to increase the adversarial loss.
4.  **Joint Evaluation & Selection**: This is the key step. We evaluate each text candidate's effectiveness against the **newly perturbed image**. This allows us to capture the cross-modal interactions. After evaluating all candidates, we select the single best text edit for that iteration.
5.  **Repeat**: We repeat this process, iteratively refining the image and text suffix until the model is successfully jailbroken.
   

---

### Putting It to the Test: Results

We tested our attack framework on several popular VLMs, and the results highlight the critical advantage of a multimodal approach.

* **Against LLaVA-RC (Robust CLIP)**: This model uses a hardened vision encoder, making it more resilient to image-only attacks. While PGD alone struggled, our joint PGD+GCG attack successfully restored a strong adversarial potency by combining weak visual perturbations with an optimized text suffix, achieving the lowest mean loss across all tested modes.
* **Against Gemma-3-4b-it (with GemmaShield2)**: This was the toughest target. Gemma's defenses were so effective that they **completely blocked image-only PGD attacks**, resulting in zero successes. However, our joint attack was still able to find a way through. By coordinating both modalities, it not only achieved a high success rate but also reduced the final adversarial loss by nearly half compared to a GCG-only text attack. This demonstrates that even when one attack surface is heavily fortified, a joint attack can amplify the remaining channels to bypass defenses.

---

### Making it Practical: Dynamic Search-Width Scheduling

A major challenge with our joint attack is that it can be computationally expensive. Evaluating every text candidate at every step takes time. To solve this, we introduced **Dynamic Search-Width Scheduling**.

The idea is to start the attack with a large budget of text candidates (a wide search) to quickly find a promising direction, and then gradually reduce the number of candidates in later iterations as the attack converges on a solution.

![](/assets/images/multimodal-attacks/search_width_comparison.png)
*Visualisation of different dynamic search-width decay schedules. This technique balances speed and attack strength.*

This simple scheduling trick works wonders. It allows us to retain nearly the full effectiveness of an expensive, wide-search attack while reclaiming most of the runtime benefits of a faster, narrow-search one. For practical use, we recommend a `256→128` schedule, which delivers excellent success rates at a manageable computational cost.

---

### Conclusion and What's Next

Our research demonstrates that joint multimodal adversarial attacks are a significant threat to modern VLMs, especially those with advanced, single-modality defenses. By perturbing image and text inputs in a coordinated fashion, these attacks can create more subtle, effective, and potentially less detectable jailbreaks than their unimodal counterparts.

The development of dynamic scheduling makes these powerful attacks practical, providing a potent tool for security researchers to audit and red-team the next generation of multimodal AI systems.

For future work, we plan to investigate the **transferability** of these attacks. If an adversarial suffix and image perturbation created for one prompt can generalize to others, it would represent an even more scalable and useful threat.

### Access the full [paper](https://github.com/Bimo99B9/BimodalAttack/blob/master/assets/report.pdf) and and code on [GitHub](https://github.com/Bimo99B9/BimodalAttack)