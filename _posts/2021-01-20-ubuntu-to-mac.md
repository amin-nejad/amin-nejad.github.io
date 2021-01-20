---
title: "Ubuntu 18.04 to macOS M1 - Developer Review"
date: 2021-01-20 00:00:00
---

# Introduction

The title says it all. This is just a short post to describe my experience of switching from Ubuntu 18.04 on an x86 machine to a MacBook Pro on Apple silicon as a machine learning engineer. My perspective is as someone who has never used a mac before so some of the below will also apply to people looking to switch to the older macs too, or similarly from an older mac to the M1.

# Specification

```diff
- Machine: Lenovo ThinkPad L390
- OS: Ubuntu 18.04.5
- Processor: Intel® Core™ i7-8565U CPU @ 1.80GHz × 8
- Memory: 16GB
- Disk: 512GB
+ Machine: MacBook Pro (13-inch)
+ OS: macOS Big Sur 11.1
+ Processor: Apple M1
+ Memory: 8GB
+ Disk: 256GB
```

# Setup

## Environment

### Terminal

Instead of using the default macOS Terminal, I would strongly recommend using [iTerm2](https://iterm2.com/) instead. It's _packed_ full of features you'd have never thought existed in a terminal.

However, regardless of which one you use, make sure to open with `Rosetta` (Rosetta 2 actually). This should install itself automatically and is responsible for translating x86 code to make it compatible with Apple Silicon. More detailed instructions can be found from the team at Notion [here](https://www.notion.so/Run-x86-Apps-including-homebrew-in-the-Terminal-on-Apple-Silicon-8350b43d97de4ce690f283277e958602).

Finally, the default shell on mac is `zsh` whereas if you are coming from Ubuntu, you will be used to the standard `bash`. `zsh` comes with a lot of cool features lacking in `bash` but if you want to stick to what you know to begin with, you can set your default shell as bash using:

```bash
chsh -s /bin/bash
```

### Package Manager

You will likely be used to `apt` but with mac you ought to switch to using `homebrew`. Assuming you have opened your terminal with Rosetta, you should be able to install `homebrew` without a hitch by following the instructions on their [website](https://brew.sh/). Its usage is pretty similar to `apt`.

### Python

macOS comes with a python 2.7 installation. If like most pythonistas, you use 3.x, you may consider uninstalling it. **Do <ins>NOT</ins> uninstall it.** I don't know why but it can break your entire OS. More information [here](https://stackoverflow.com/questions/3819449/how-to-uninstall-python-2-7-on-a-mac-os-x-10-6-4). Just leave it be and set up whatever python you want in virtual environments.

On that note, many people have reported problems getting `pyenv` (or virtual environments in general) to work probably on the M1, and consequently libraries installed in the environment. I've somehow managed to avoid this entirely by using `conda` for my virtual environments. I tend not to install packages with `conda` if I can avoid it though (despite their recommendations) and just use `pip` within the `conda` environment. I haven't had any problems with my environment or any libraries as a consequence.

### Deep learning libraries

Assuming you've followed what I've done above, you should be able to install `tensorflow` and `pytorch` without a problem using `pip` as standard. However, their performance is another matter indeed.

#### **Tensorflow**

Apple has kindly provided a [fork](https://github.com/apple/tensorflow_macos) of `tensorflow` which has been adapted to run on Apple silicon natively without needing Rosetta. If you install this version of `tensorflow` (instead of using pip as described above), you should be able to enjoy the significant speedups that the M1 offers.

You can find various posts online benchmarking M1 performance but the best one I've come across is [this one](https://towardsdatascience.com/benchmark-m1-vs-xeon-vs-core-i5-vs-k80-and-t4-e3802f27421c) which compares Apple's `tensorflow` fork against regular `tensorflow` on GPUs. The chart below(reproduced from the benchmarking blog post) shows how the M1 compares against Nvidia's K80 and T4 GPUs at training 3 different models using 3 different batch sizes.

<img class="blog_image" src="/images/m1-tensorflow.png">

We learn a few things about the M1 from this chart compared to the Nvidia GPUs:

- it does much better at smaller batch sizes significantly beating the K80 and rivalling the T4
- the performance boost at higher batch sizes is very gradualmuch smaller e.g. increasing the batch size by 4 orders of magnitude (32 to 512) training the LSTM only offers a 3x speedup for the M1 but both the K80 and T4 get a 6X speedup
- it performs comparatively worse at convolutions where it is actually significantly slower than the K80 at larger batch sizes

#### **Pytorch**

Unfortunately the situation is rather different with `pytorch`. There is no Apple fork and `pytorch` itself does not yet support Apple silicon so we are stuck with using Rosetta which significantly hampers our performance. There is a lot of discussion about this on github, [this](https://github.com/pytorch/pytorch/issues/47702) is the best thread to follow going forward.

I don't have any official benchmark stats but in my experience so far, `pytorch` performance on the M1 is comparable to my previous machine. If anything, it might actually be a little slower but nothing significant.

## VS Code

I use VS Code as my IDE the vast majority of the time. This doesn't support M1 yet either and so is actually noticeably slower than my previous machine. Not frustratingly so, just noticeably so. However support has been introduced in the bleeding edge version known as [VS Code Insiders](https://code.visualstudio.com/insiders/#osx). Having tried it out briefly, I didn't notice much of a difference so decided to stay with the stable release and just wait for it to be supported.

A few issues I've encountered so far:

- the integrated terminal is stuck on the aforementioned system python 2.7. This appears to be an issue on older macs too. Fix for it [here](https://stackoverflow.com/questions/54582361/vscode-terminal-shows-incorrect-python-version-and-path-launching-terminal-from). (You just need to modify the `settings.json`)
- python `multiprocessing` doesn't appear to work properly (it crashes with weird logs) when using the integrated terminal only - it works fine when the script that runs it is called from the regular terminal though. I'm not sure what the exact problem is but I know I didn't have it using Linux
- The extensions marketplace was completely empty showing nothing. Not sure what was going on here, eventually I just rebooted which solved the problem

## Browser

The situation is quite good here. Chrome and others (e.g. [Brave](https://brave.com/)) introduced support very quickly and the difference is HUGE compared with the x86 version on M1 (and probably slightly quicker than the x86 version on my old machine too). I use Brave and the ARM version on M1 is just as quick as the native Safari that came with the machine.

## Other

- Ubuntu 18.04 comes installed with OpenJDK version 11 which is handy. This is not the case on mac and needs to be installed separately
- Git autocomplete does work out of the box. Fix [here](https://stackoverflow.com/questions/14970728/homebrew-s-git-not-using-completion)
- No `wget` out of the box, needs to be installed separately

# Miscellaneous

Nothing to do with developing really but it's worth mentioning how good:

- the battery life is. I can easily work a full day on a single charge and still have plenty of battery left at the end of it. It's unbelievable. The fan is extremely quiet too and it doesn't get hot either.

- the display is. The 4k retina display is almost too indulgent. Not really a game changer but it's a nice-to-have

- [Alfred](https://www.alfredapp.com/) is. It's a productivity app which just makes your life easier only available on Mac

# Final thoughts

Hopefully the above should help you hit the ground running dev-wise if you're just getting started on the M1. Thankfully my experience has been pretty positive so far, especially given that there are always teething issues for early-adopters.

However if you're still thinking about taking the plunge, I would probably recommend holding off a few months as bugs get ironed out and more applications/libraries (particularly `pytorch`) being to support the ARM hardware natively, since using Rosetta does significantly constrain M1 performance.
