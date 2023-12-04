---
title: The Tech Behind TikTok's Addictive Recommendation System
date: 2023-12-04 21:30 +0200
categories: [TikTok, Recommendation Systems, Kafka, Flink]
tags: [tiktok, recommendation systems, kafka, flink]
author: tremo
---
## Intro

I’ve been using TikTok a lot lately and was curious about the tech behind TikTok’s addictive algorithm, the "Recommendation System." I visited their official [blog post](https://newsroom.tiktok.com/en-us/how-tiktok-recommends-videos-for-you) but found limited details on how it works. So, I googled and found two papers “[Monolith](https://arxiv.org/pdf/2209.07663.pdf)” and “[IPS](https://www.cs.princeton.edu/courses/archive/spring21/cos598D/icde_2021_camera_ready.pdf),” released by ByteDance, TikTok’s parent company, and here is the process as explained by them.

## Recommendation features

There are features that influence the recommendation system, which are:

**User Features:**

- User interactions, such as the videos you like, share, comment on, watch in full or skip, accounts you follow, accounts that follow you, and when you create content.
- User information, such as your device and account settings, language preference, country, time zone and day, and device type.

**Content Features:**

- Video information, such as captions, sounds, hashtags, number of video views, and the country in which the video was published.

These elements are merged to create detailed user profiles and content embeddings. These profiles and embeddings are then used to tailor content suggestions using a mix of collaborative and content-based approaches.

## Model architecture

Embeddings are input to a Deep Factorization Machine, which leverages a deep neural network to capture non-linear and complex patterns from the embeddings, and Factorization Machines to capture linear correlations between different features, which is essential in understanding simple yet effective user-item interactions.

![Model Architecture](/assets/img/posts/2023-12-04-the-tech-behind-tiktoks-addicitve-recommendation-system/model_architecture.png)
_Model Architecture_

## Real-Time Online Training

TikTok has to update its recommendation system as fast as possible to account for the non-stationary nature of user data, known as “Concept Drift.” So, there has to be a mechanism that updates the model parameters in real-time.

“Monolith” framework uses a streaming engine that can be used in both batch and online training in a unified design. They use a Kafka queue to log actions of users (clicks, likes, comments, etc.) and another Kafka queue for features. And Flink as a streaming engine/job for joining them to create training examples.

During online training, sparse parameters are updated on a minute-level interval from the training PS (Parameter Synchronization) to the serving PS, which avoids heavy network transmissions or memory spikes.

![Training Pipeline](/assets/img/posts/2023-12-04-the-tech-behind-tiktoks-addicitve-recommendation-system/training_pipeline.png)
_Training Pipeline_

Monolith uses TensorFlow’s distributed Worker-ParameterServer, where multiple machines work together to train a model. Workers are responsible for performing computations (gradients/parameter updates), while parameter servers are used for storing the current model state like weights and biases.

![TensorFlow Distributed System](/assets/img/posts/2023-12-04-the-tech-behind-tiktoks-addicitve-recommendation-system/worker-ps-architecture.png)

## Managing Large and Dynamic User Data: Embedding Collisions

User data's vast and dynamic nature poses a challenge, as it can lead to unwieldy embedding sizes and collisions (where different data points are mistakenly identified as identical).

To tackle this, "Monolith" employs "cuckoo hashing." This method uses two hash tables with separate hash functions, allowing dynamic placement and repositioning of elements to avoid collisions.

Additionally, to prevent the embedding memory from expanding too rapidly, a probabilistic filter and an adjustable expiration period for embeddings are used, effectively managing the memory requirements.

![Cuckoo Hashing](/assets/img/posts/2023-12-04-the-tech-behind-tiktoks-addicitve-recommendation-system//cuckoo_hashmap.png)

## Conclusion

TikTok’s recommendation system played a main role in its success and widespread use. I tried in this blog post to shed some light on the underlying technologies used, especially the online training, which helps them recommend real-time personalized content that keeps users staring at their phones for hours.

**References:**

1. [How TikTok recommends content](https://support.tiktok.com/en/using-tiktok/exploring-videos/how-tiktok-recommends-content)
2. [Monolith: Real Time Recommendation System With Collisionless Embedding Table (arxiv.org)](https://arxiv.org/pdf/2209.07663.pdf)
3. [*An Empirical Investigation of Personalization Factors on TikTok (arxiv.org)](https://arxiv.org/pdf/2201.12271v1.pdf)
4. [cs.princeton.edu/courses/archive/spring21/cos598D/icde_2021_camera_ready.pdf](https://www.cs.princeton.edu/courses/archive/spring21/cos598D/icde_2021_camera_ready.pdf)
