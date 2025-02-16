---
layout: post
title: OSEE Course and Exam Review
subtitle:
cover-img: /assets/img/osee_blog_header.jpg
thumbnail-img: /assets/img/osee_logo.webp
share-img: /assets/img/osee_logo.webp
tags: [OSEE, EXP-401, AWE]
excerpt:
---

## Introduction

Many blogs detail the path to achieving OSEE certification. Do we need another? Maybe not. Yet, after completing my exam, I felt that many of these reviews painted an unnecessarily intimidating picture.

This blog aims to reassure future students: OSEE is absolutely attainable. The hardest part? Convincing your CTO to approve the extortionate training fee!

## The Course

The OSEE course is often considered the most challenging offensive security training available. This reputation can make students doubt if they can keep up with the instructor, let alone even think about passing the exam.

However, despite its technical depth, the teaching is outstanding. I particularly appreciated the instructor's approach of building each exploit from scratch, explaining their thought process along the way. Every step is detailed in the accompanying book, and questions are always welcome — the instructors enjoy taking on tangents sparked by student questions. This step-by-step structure makes the book an invaluable resource for future reference.

The vulnerabilities covered were once real-world issues. While they aren’t especially complex (given the need to write a full exploit in a day), they serve as excellent examples and give students hands-on experience with incredibly important attack surfaces.

If you have experience with Windows security research and are comfortable with a debugger, you’ll be fine! Familiarity with DEP/ASLR and ROP are probably the only real prerequisites.

I particularly enjoyed the Extra Mile challenges that allowed students to complete additional tasks in the evenings. The challenge coin reward is a nice touch and definitely helped to solidify that day's learning. They take a few hours each.

## The Exam

### Exam Preparation

It's made very clear by the [OSEE Exam Report template](https://www.offsec.com/awe/AWE-Exam-Report.docx) that the exam is split into two halves - Kernel and Browser. To prepare for the exam, I focused mainly on the Browser module. I've spent many years working with the Windows kernel and had never touched browser exploits before, so split my time around 90:10. 

#### Browser

I spent a very long time implementing all the concepts covered in the course in the form of a Browser Exploitation Framework. The idea was that I could then swap in and practice other techniques, such as a different method of calling WINAPIs or a different sandbox escape. I learned lots writing it and it proved incredibly useful for the final exam as I was very comfortable with bypassing all mitigations in a few different ways.

The framework is on [GitHub](https://github.com/ntdelta/chakra-exploit-framework) if you fancy taking a look. I'd say I relied more on [Connor Mcgarr's blogs](https://connormcgarr.github.io/type-confusion-part-2/) than the AWE book as I felt the explanations were better (and I hate carrying an 800 page book around with me). When preparing, I'd recommend focusing on becoming comfortable taking a goal, such as 'implement the leafInterpreterFrame CFG bypass', yourself with limited reliance on online tutorials. It is easy to become reliant on turning to the next page to copy out the next step of the exploit. If you just do this, you will not be able to pass the exam.

The Extra Miles are also worth doing, especially the initial RCE for the other Type Confusion CVEs. Not just because they'll help with the exam, but because they will make you a much better exploit developer - thats why we signed up to the course after all!

#### Kernel

As mentioned previously, I spend a lot of time messing with the Windows kernel and therefore didn't allocate as much time to this. Once again, I wrote a Kernel Exploit Framework which implemented the key techniques covered in the course. It was written in a way that allows you to plug a `Write` and a `Read` primitive in and achieve privilege escalation in a variety of ways. Again, the framework approach meant I could pick and choose CVEs and Extra Miles to implement quickly without having to start from scratch each time. Unfortunately, I feel the code resembles that of the AWE material too closely to publish online but these were the key classes:

```cpp
class ROPChain {
private:
	std::vector<uint64_t> chain;

public:
	void addGadget(uint64_t address) {
		chain.push_back(address);
	}

	void write_to_buffer(uint8_t* buffer) {}

	void printChain() {
		for (auto& addr : chain) {
			std::cout << "Gadget address: 0x" << std::hex << addr << std::dec << std::endl;
		}
	}

	size_t size() const {
		return chain.size();
	}
};

class KernelExecFramework {
public:
	using WriteFunc = std::function<void(void*, void*, uint32_t)>;
	using ReadFunc = std::function<void(void*, void*, uint32_t)>;

	KernelExecFramework(WriteFunc writeFunc, ReadFunc readFunc)
		: write(writeFunc), read(readFunc) {}

	void perform_write(void* dest, void* source, uint32_t len) {
		if (write) {
			write(dest, source, len);
		}
	}

	void perform_read(void* read_from, void* read_to, uint32_t len) {
		if (read) {
			read(read_from, read_to, len);
		}
	}

	uint64_t read_qword(void* address) {
		uint64_t value = 0;
		perform_read(address, &value, sizeof(value));
		return value;
	}

	void write_qword(void* address, uint64_t value) {
		perform_write(address, &value, sizeof(value));
	}

	void bypass_smep(void* address) {}

private:
	WriteFunc write;
	ReadFunc read;
};

cont.
```

You can then implement the read/write for a given vulnerability and plug it straight in:
```cpp
ExampleExploit example_exploit(driver_handle, module_base);
KernelExecFramework kernel_exec_framework(
	[&](void* dest, void* source, uint32_t len) { example_exploit.write_memory(dest, source, len); },
	[&](void* dest, void* source, uint32_t len) { example_exploit.read_memory(dest, source, len); }
);
```

I also recommend completing the `6.7.3.2` Extra Mile.

### Exam Day

#### Timings

Day 1:
```
06:00 Exam Start.
12:30 Flag 1 submitted. Start lunch.
13:30 Resume. Start 2nd question.
22:00 Sleep.
```

Day 2:
```
09:30 Resume.
14:00 Submit 2nd flag.
17:00 Submit Report. End Exam.
```

### Exam Review

Overall, the exam was far easier than I was expecting. The challenges were all straightforward to complete. Each stage of the exploit writing process led nicely into the next part of the question and there was never a point where I was stuck for what to do next. One question was far easier than the other, I would recommend starting with this first - it'll be obvious. It will help you build confidence and secure you 50 points.

It's really hard to say anything more than that. Just know that if you've completed the Extra Miles, and feel very confident in WinDbg, you'll do great.

### Infrastructure Tip

The VPN requires you to connect from Kali. This was annoying as RDP to VM->RDP to Exam Box is a gross workflow. So, I set up a Kali VM on my host and connected to the OffSec VPN from the VM. I then used the `-D` SOCKS forwarder and the conventional `-L:1337:...:3389` port forwarding flags to SSH into the VM from my host. This meant I could RDP to the OffSec VM directly from my host using `127.0.0.1:1337`. It was a much nicer workflow, would highly recommend.

## Conclusion

To conclude, I feel the AWE course is perfect for people who are fresh off of OSED and are looking for their next challenge. It is also great for people who do security research, maybe malware analysis, and are looking to see into the world of "proper" exploit development.

The teaching is great and the course is well thought out. Despite this, I do not think it is worth the price tag. Resources such as Connor's blogs, OffByOne's streams, and ol' reliable Unknown Cheats, are all free and cover most of the content. I think it is a shame they do not offer a "learn at your own pace" OSED style subscription at a more affordable price point.

If you can get your employer to pay for it, do not be put off by it's reputation. Remember that at the end of the day, it is in the interest of those who hold the OSEE certification to exaggerate its difficulty - it's on their CV now, after all!
