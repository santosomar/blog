---
title: "Accelerating Llama models with AWS Inferentia2"
thumbnail: /blog/assets/TODO/thumbnail.png
authors:
- user: dacorvo
---

# Accelerating Llama models with AWS Inferentia2

In a [previous post on the Hugging Face blog](https://huggingface.co/blog/accelerate-transformers-with-inferentia2), we introduced [AWS Inferentia 2](https://aws.amazon.com/ec2/instance-types/inf2/), the second-generation AWS Inferentia accelerator, and explained how you could use [optimum-neuron](https://huggingface.co/docs/optimum-neuron/index) to quickly deploy Hugging Face models for standard text and vision tasks on AWS Inferencia 2 instances.

In a further step of of integration with the [AWS Neuron SDK](https://github.com/aws-neuron/aws-neuron-sdk), it is now possible to use 🤗 [optimum-neuron](https://huggingface.co/docs/optimum-neuron/index) to deploy LLM models for text generation on AWS Inferentia 2.

And what better model could we choose for that demonstration than [llama2](https://huggingface.co/meta-llama/Llama-2-13b-hf), one of the most popular model on the [Hugging Face hub](https://huggingface.co/models).

## Export the Llama2 model to Neuron

As explained in the [optimum-neuron documentation](https://huggingface.co/docs/optimum-neuron/guides/export_model#why-compile-to-neuron-model), models need to be compiled and exported to a serialized format before running them on Neuron devices.

Fortunately, 🤗 `optimum-neuron` offers a [very simple API](https://huggingface.co/docs/optimum-neuron/guides/models#configuring-the-export-of-a-generative-model) to export standard 🤗 [transformers models](https://huggingface.co/docs/transformers/index) to the Neuron format.

```
>>> from optimum.neuron import NeuronModelForCausalLM

>>> compiler_args = {"num_cores": 1, "auto_cast_type": 'fp16'}
>>> input_shapes = {"batch_size": 1, "sequence_length": 2048}
>>> model = NeuronModelForCausalLM.from_pretrained(
        "meta-llama/Llama-2-13b-hf",
        export=True,
        **compiler_args,
        **input_shapes)
```

This deserves a little explaination:
- using `compiler_args`, we specify on how many cores we want the model to be deployed (each neuron device has two cores), and with which precision (here `float16`),
- using `input_shape`, we set the static input and output dimensions of the model. All model compilers require static shapes, and neuron makes no exception. Note that the
`sequence_length` not only constrains the length of the input context, but also the length of the KV cache, and thus, the output length.

Depending on your choice of parameters and inferentia host, this may take from a few minutes to more than an hour.

Fortunately, you will need to do this only once because you can save your model and reload it later.

```
>>> model.save_pretrained("a_local_path_for_compiled_neuron_model")
```

Even better, you can push it to the # Push the neuron model to the [Hugging Face hub](https://huggingface.co/models).

```
>>> model.push_to_hub(
        "a_local_path_for_compiled_neuron_model",
        repository_id="my-neuron-repo",
        use_auth_token=True)
```

## Generate text using a neuron model on AWS Inferentia 2

Once your model has been exported, you can generate text using the transformers library, as it has been described in [details in this previous post](https://huggingface.co/blog/how-to-generate).

```
>>> from transformers import AutoTokenizer

>>> tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-2-13b-hf")
>>> tokenizer.pad_token_id = tokenizer.eos_token_id
>>> tokenizer.padding_side = "left"

>>> inputs = tokenizer("What is deep-learning ?", return_tensors="pt", padding=True)
>>> outputs = model.generate(**inputs,
                             max_new_tokens=128,
                             do_sample=True,
                             temperature=0.9,
                             top_k=50,
                             top_p=0.9)
>>> tokenizer.batch_decode(outputs, skip_special_tokens=True)
['<s> What is deep-learning ?\nThe term “deep-learning” refers to a type of machine-learning
that aims to model high-level abstractions of the data in the form of a hierarchy of multiple
layers of increasingly complex processing nodes.']
```

Note however that a few restrictions apply.

The following generation strategies are supported:

- greedy search,
- multinomial sampling with top-k, top-p with temperature.

Most logits pre-processing/filters (such as repetition penalty) are supported.

## All-in-one with `optimum-neuron` pipelines

For those who like to keep it simple, there is an even simpler way to use an LLM model on AWS inferentia 2 using [optimum-neuron pipelines](https://huggingface.co/docs/optimum-neuron/guides/pipelines).


Using them is as simple as:

```
>>> from optimum.neuron import pipeline

>>> p = pipeline('text-generation', 'dacorvo/Llama-2-7b-hf-neuron-budget')
>>> p("My favorite place on earth is", max_new_tokens=64, do_sample=True, top_k=50)
[{'generated_text': 'My favorite place on earth is the ocean. It is where I feel most
at peace. I love to travel and see new places. I have a'}]
```

## Benchmarks

But how much efficient is text-generation on Inferentia 2 ?  Let's figure out !

We have uploaded on the hub pre-compiled versions of the LLama2 7B and 13B models with different configurations:

| Model type              | num cores | batch_size | Hugging Face Hub model               |
|-------------------------|-----------|------------|--------------------------------------|
| Llama2 7B - budget      | 2         | 1          |dacorvo/Llama-2-7b-hf-neuron-budget   |
| Llama2 7B - latency     | 24        | 1          |dacorvo/Llama-2-7b-hf-neuron-latency  |
| Llama2 13B - latency    | 24        | 1          |dacorvo/Llama-2-13b-hf-neuron-latency |
| Llama2 13B - throughput | 24        | 4          |dacorvo/Llama-2-13b-hf-neuron-tgi     |

> Note: the models compiled for 24 cores can only run on an `inf2.48xlarge` instance.

To evaluate the models, we generate tokens up to a sequence length of 1024, starting from
input tokens corresponding to 25, 50 and 75 % of the sequence length (i.e. 256, 512 and 768).

### Latency

The latency corresponds to the total time to reach a sequence length of 1024 tokens.

| Model / input ratio    | 25 %   | 50 %   | 75 %   |
|------------------------|--------|--------|--------|
| Llama 7B - budget      | 47.4 s | 31.9 s | 16.2 s |
| Llama 7B - latency     | 6.7 s  | 4.8 s  | 3 s    |
| Llama 13B - latency    | 10.4 s | 7.4 s  | 5 s    |
| Llama 13B - throughput | 13 s   | 10.7 s | 9.6 s  |

As you can see, the latency is not that much impacted by the increased batch size for the 13B model.

### Throughput

The throughput corresponds to the average number of tokens generated by second.
It is obtained by dividing the total time to reach a sequence length of 1024 tokens
by the total number of output tokens (i.e batch_size * 1024).

| Model / input ratio    | 25 %    | 50 %    | 75 %    |
|------------------------|---------|---------|---------|
| Llama 7B - budget      | 22 t/s  | 32 t/s  | 63 t/s  |
| Llama 7B - latency     | 153 t/s | 213 t/s | 341 t/s |
| Llama 13B - latency    | 98 t/s  | 139 t/s | 206 t/s |
| Llama 13B - throughput | 315 t/s | 383 t/s | 425 t/s |

