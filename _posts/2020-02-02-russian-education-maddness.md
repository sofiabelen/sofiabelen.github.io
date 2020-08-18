---
layout: post
title: "Russian education madness: Learning more than 60 theorems with their proofs in your first semester"
categories: blog
image: /images/mipt2.jpg
preview: "How to survive in a Russian university as a foreign student."
redirect_from:
  - /selfstudy/Russian-education-maddness.html
description: "My first semester in a Russian university. How anki flashcards helped me study over 60 theorems with their proofs for final exam."
thumbnail: lk-cropped.jpg
---

<img style='height: 100%; width: 100%; object-fit: contain' src="/images/lk.jpg" atl="Laboratory Corpus MIPT.">

If you're a PhysTech student, you know what I'm talking about when I say the first semester is HARD. If you're also a foreign student like me, who had a year to learn the language before starting on one of the hardest universities in the world... Then I feel you. Want to know how I made it out alive the first semester? Then keep reading.

If you don't know what **PhysTech** is, it's the informal name for **MIPT** (Moscow Institute of Physics and Technology), one of the leading research universities of Russia. It was funded during the Soviet Union, with the goal to select and train the most gifted students of the country. During the first three years of a bachelor's program, you study the core subjects at an extremely intensive rate, in order to prepare you to start working on research as early as your possible, sometimes as early as on your second year. Also, a huge importance is given to competitions and olympiads.

You might be curious to learn more about what it's like studying in Russia, from the perspective of a foreign student. Also, I want to share what I consider a pretty good **studying technique**, that comes in handy when you have to learn an enormous amount of material, and can be tweaked for probably any subject.

As a side note, sorry if this is messy, I didn't have much time to edit, but I thought this might help others who are in the same situation.

Here are some quick links for getting around:

> [Russian way of teaching math](#russian-way-of-teaching-math)
> 
> [Spaced repetition software: a lifesaver](#spaced-repetition-sofware-a-lifesaver)
> 
> [Anki and how to study your flashcards](#anki-and-how-to-study-your-flashcards)
> 
> [1. Pro mode: Typesetting with Latex](#pro-mode-typesetting-with-latex)
> 
> [2. Normal mode: With pictures of your current summaries](#normal-mode-with-pictures-of-your-current-summaries)
> 
> [3. Express mode: Just take screenshots of your textbook](#express-mode-just-take-screenshots-of-your-textbook)

## Russian way of teaching math

<figure>
<img style='height: 85%; width: 85%; object-fit: contain' src="/images/mgu.jpg" atl="Moscow State University">
  <figcaption>Moscow State University</figcaption>
</figure>
&nbsp;  

What intrigued me most about the Russian culture was their education system. How did they manage to always come up top in all *'intellectual' competitions*, like **ACM-ICPC**? I knew I could train for 10 years straight and still not be at their level. I read many opinions on the internet on this topic, but I wanted to see for myself what their secret was.

> "Going to PhysTech is like going to the **army**: you acquire some lifesaving skills along the way, and it's kind of a survival of the fittest."

It wasn't until I started my first year here in PhysTech that I knew. Their approach to subjects like mathematics, physics and programming is radically different from what I knew before. In physics and programming, basically, you're expected to be a problem solving machine. In fact, their unique problem-solving approach is what initially drew me to physics, [more on that here](/mystory/from_programming_to_physics.html). However, now we're going to talk about mathematics, more specifically **mathematical analysis**.

In Russia, on the first semester of any science <img style='height: 50%; width: 50%; object-fit: contain' align="right" src="/images/mipt.jpg" atl="MIPT">related degree, one of the toughest subjects you'll have is mathematical analysis. **'Oh, calculus!'**, you might think. Well, think again... No one cares if you know how to take derivatives, integrate, or take limits, that's high school material. Instead, you're here to learn all the theorems and proofs behind that. Taking limits by using Taylor formulas should become second nature. As should be evaluating the n-th derivative, when it's possible. By the end of the first semester, you have to know by heart over 60 theorems and proofs, all the way from 'Cantor's theorem of nested intervals' to 'Taylor's theorem with Peano's and Lagrange's form of remainder'. At the end of the term we take a written and an oral exam. The oral is purely theoretical, and has twice the weight on your final grade than the written. Needless to say, it's also the hardest. It basically goes like this, you pull out an exam ticket (билет) at random, with the name of two theorems. You are given some time to write out their proof, with all corresponding definitions and lemmas, and then a professor calls you. He/she reads what you wrote and ask questions to make sure you understand it. Afterwards, they will ask you extra questions about any other theorem or give you a practical exercise to solve. If you haven't been sent to retake(our infamous пересдача) by this time, you're the happiest person on earth.

> "No one cares if you know how to take derivatives, integrate, or take limits, that's high school material."

<img style='height: 50%; width: 50%; object-fit: contain' align="right" src="/images/mipt2.jpg" atl="MIPT">During the semester, I felt like I was always behind, since I didn't have the necessary background that people here get during school, and I had to spent half my time translating before I could get to actually studying. I always hesitated to ask questions because I just didn't know how to say it in Russian. It was also hard to know what to prioritize and what I shouldn't be wasting my time on, since I didn't really understand how the system worked. So, it was an uphill battle, and I know I'm not the only one. Going to PhysTech is like going to the army: you acquire some lifesaving skills along the way, and it's kind of a survival of the fittest. And I'm not even talking about intelligence. You need a mixture of **perseverance** and out-of-this-world **stress + time management** skills. Also, you should try not to isolate yourself too much, since you might miss out some crucial piece of information.

Okay, so here it goes, the one thing that truly helped me on my first semester.

## Spaced repetition software: a lifesaver

There is a bit of a controversy here, as many will argue that learning a proof by memorizing it is useless. And I couldn't agree more, this is why you have to be careful, and not fall in the trap of rote memorizing stuff. Remember, understanding always comes first. This is crucial.

> "Our minds are much more likely to remember a set of logical steps than a random recompilation of symbols."

For those of you that haven't heard about SRS, <img style='height: 25%; width: 25%; object-fit: contain' align="right" src="/images/anki.png" atl="Anki logo">on the language learning community, we call it flashcards on steroids. The idea is to take advantage of how our brains work and review the material just as you're about to forget it. Of course, no algorithm can guess this exact moment, but it can make some pretty approximations. It was my main method for learning vocab as I studied German and then Russian, and I thought, "couldn't I tweak it to make it work for theorems and proofs?"

I'm sure everyone will agree with me on this: in order to learn some proof, first of all, you have to **understand** it. Then, you have to write it down at least a few times, each time you'll be able to recall more and more without peaking at your textbook. And there will be parts that are harder to remember than others. But it's not very efficient to sit down and spend the whole afternoon writing a proof again and again.

So, the basic idea of SRS is that you'll be reviewing more often the concepts that are **toughest** to recall, so you don't waste time studying what you already know. This is very straight forward, for example, for learning vocab, but how do you translate this to learning math? The trick is to train your brain to remember the logic behind a certain concept, or how to derive a formula, rather than just remembering the formula itself. That way, when the exam day comes, you won't have to rely on your memory to remember all that material! That would most likely fail you...taking into account all the stress and nerves on that day. 

Okay, how does this work? Let's say the flashcard for a certain formula comes up and you don't remember it, you shouldn't look at the answer right away. Instead, you should try to derive it from what you already know, that way, you're not relying on memory but rather on logic to recall material. Our minds are much more likely to remember a set of logical steps than a random recompilation of symbols. That's the most important thing to keep in mind, and why there's no use in trying to memorize without understanding. Another key is to summarize as much as you can, so you don't waste time on learning clutter.

## Anki and how to study your flashcards

<p>
<img style='height: 100%; width: 100%; object-fit: contain' src="/images/Flashcards6.png" atl="Example of cloze-deletion">
<em>Back and front of an Anki flashcard using cloze deletion.</em>
</p>

Anki is this magical piece of software that I've been using for years now and has come to help me once again.

How I usually study, is by writing down on paper what I remember. Even if you can't remember everything, try to write something, and give yourself time to think before looking at the answer. This is a very important part of process, since you're replicating an exam. You have to teach your brain to recall the logical steps associated with each theorem.

On the topic of actually making your flashcards, I'd say there are basically three ways, depending on how much time you count on, which I will talk about bellow. For language learning, we usually use the normal back-front/question-answer model, but in this case, anki has a feature that comes in handy: **cloze deletion**. This is basically hiding part of a text, and that's what you have to recall. You can have more than one of those in a single card, this way you can divide a proof into smaller chunks that are easier to digest.

Tip: use AnkiDroid for studying in your phone. This is great for making use of small windows of time throughout your day. However, your main studying should be done in your desk, with a piece of paper and pen.

## 1. Pro mode: Typesetting with Latex

<img style='height: 100%; width: 100%; object-fit: contain' src="/images/Flashcards2.png" atl="Example of a card using method 2">

This is somewhat time consuming, but it's definitely the most optimal. It forces you to summarize and understand everything before creating a card, so reviewing should go more smoothly. In a way, you are investing time at the beginning, but it then pays off with the reduced review time(compared to the other methods).

The way you create your cards is: you go to 'Add', then 'Cards', and copy+paste the code bellow on 'Back template', 'Styling', and 'Front template', respectively. It's a script to have mathjax compile your latex code, plus the color scheme that I use. Where I got the [__css styling__](https://medshamim.com/med/how-to-design-beautiful-anki-cards) and the [__mathjax script__](https://www.reddit.com/r/Anki/comments/54c967/how_to_use_mathjax_in_anki_ankidroid_online_and/).

<details>

<summary>Back template</summary>

~~~~
<div id="kard">
{{cloze:Text}}
</div>

<script type="text/x-mathjax-config">
MathJax.Hub.processSectionDelay = 0;
MathJax.Hub.Config({
  messageStyle: 'none',
  showProcessingMessages: true,
  tex2jax: {
    inlineMath: [['$', '$']],
    displayMath: [['$$', '$$']],
    processEscapes: true
  }
});
</script>
<script type="text/javascript">
(function() {
  if (window.MathJax != null) {
    var card = document.querySelector('.card');
    MathJax.Hub.Queue(['Typeset', MathJax.Hub, card]);
    return;
  }
  var script = document.createElement('script');
  script.type = 'text/javascript';
  script.src = 'https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_SVG';
  document.body.appendChild(script);
})();
</script>
~~~~

</details>

<details>

<summary>Styling</summary>

~~~~

html { overflow: scroll; overflow-x: hidden; }
/* CONTAINER FOR YOUR CARDS */
#kard {
    padding: 0px 0px;
    max-width: 700px; /* CHANGE CARD SIZE HERE */
    margin: 0 auto; /*CENTERS THE CARD IN THE MIDDLE OF THE WINDOW */
}

/* APPLIES TO THE WHOLE CARD */
.card {
    font-family: Menlo, baskerville, sans;
    font-size: 18px; /* FONT SIZE */
    text-align: left; /* ALIGN TEXT */
    color: #D7DEE9; /* FONT COLOR */
    line-height: 1.6em;
    background-color: #333B45; /* BACKGROUND COLOR */
}
/* STYLE FOR CLOZE DELETIONS */
.night_mode .cloze, .cloze b, .cloze u, .cloze i { 
font-family: Menlo, baskerville, sans;
font-size: 18px; /* FONT SIZE */
line-height: 1.6em;
color: MediumSeaGreen !important;}

/* STYLE FOR EXTRA PORTION ON BACK OF CARD */
#extra, #extra i { font-size: 15px; color:#D7DEE9; font-style: italic; }

/* STYLE TAGS TO APPEAR WHEN HOVERING OVER TOP OF CARD */
.tags { 
    color: #A6ABB9;
    opacity: 0;
    font-size: 10px; 
    width: 100%;
    text-align: center;
    text-transform: uppercase; 
    position: fixed; 
    padding: 0; 
    top:0;  
    right: 0;}
.tags:hover { opacity: 1; position: fixed;}

/* IMAGE STYLE */
img { display: block; max-width: 100%; max-height: none; margin-left: auto; margin: 10px auto 10px auto;}
tr {font-size: 12px; }

/* COLOR ACCENTS FOR BOLD-ITALICS-UNDERLINE */
b { color: #C695C6 !important; } /* BOLD STYLE */
u { text-decoration: none; color: #5EB3B3;} /* UNDERLINE STYLE */
i  { color: IndianRed; } /* ITALICS STYLE */
a { color: LightGray !important; text-decoration: none; font-size: 10px; font-style: normal; } /* LINK STYLE */

/* ADJUSTMENT FOR MOBILE DEVICES */
.mobile .card { color: #D7DEE9; background-color: #333B45; } 
.mobile .tags:hover { opacity: 1; position: relative;}
~~~~

</details>


<details>

<summary>Front template</summary>

~~~~
<div id="kard">
{{cloze:Text}}<br>
{{Extra}}
</div>

<script type="text/x-mathjax-config">
MathJax.Hub.processSectionDelay = 0;
MathJax.Hub.Config({
  messageStyle: 'none',
  showProcessingMessages: true,
  tex2jax: {
    inlineMath: [['$', '$']],
    displayMath: [['$$', '$$']],
    processEscapes: true
  }
});
</script>
<script type="text/javascript">
(function() {
  if (window.MathJax != null) {
    var card = document.querySelector('.card');
    MathJax.Hub.Queue(['Typeset', MathJax.Hub, card]);
    return;
  }
  var script = document.createElement('script');
  script.type = 'text/javascript';
  script.src = 'https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-MML-AM_SVG';
  document.body.appendChild(script);
})();
</script>
~~~~

</details>


<details><summary>Example of a longer card</summary>
<img style='height: 100%; width: 100%; object-fit: contain' src="/images/Flashcards4.png" atl="Example of a card">
</details>
&nbsp;  

To write latex, you just write it between '\\( \\)'. What you want to be inside a cloze deletion should be inside \{\{c1:: \}\}, change c1 to c2 for creating a second cloze. Quick example:

<img src="/images/Flashcards8.png" atl="Example of a card.">

Which will translate to:

<img style='height: 100%; width: 100%; object-fit: contain' src="/images/Flashcards7.png" atl="Example of a card.">


## 2. Normal mode: With pictures of your current summaries

<img style='height: 100%; width: 100%; object-fit: contain' src="/images/Flashcards1.png" atl="Example of a card using method 1">

If you already have everything written down nicely, ready to study, I'd recommend this method. Just take pictures of everything, crop them into smaller sections (use some image editor or just do it on your phone), add them to anki as separate clozes, and you're ready to start studying. For example, you can divide a card into: definitions, formulation of the theorem, and proof. If the proof has more than one subsection, take advantage of this, and put each on a different cloze. The trick is to try to make each cloze as small as possible. That will make studying less stressful and more efficient. 

Also don't worry if your cards don't look the prettiest, they're not supposed to.

## 3. Express mode: Just take screenshots of your textbook

<p>
<img style='height: 100%; width: 100%; object-fit: contain' src="/images/Flashcards3.png" atl="Example of a card using method 3">
<em>Screenshots taken from: Иванов Г.Е. Лекции по математическому анализу</em>
</p>

I'm guilty of abusing this, especially when I knew I didn't have enough time. However, I found this to be the least effective of the three. The other two methods force you to understand and write it with your own words before you make a card. That means that each card that is added is not something completely new. But if you're just taking screenshots of some external source (books or websites), you're not sure you understand it completely before adding it. Also, the information won't be as concise, and clutter doesn't help when learning. Remember, you have to be efficient, so by keeping the information to a minimum, you ensure you're not wasting precious study minutes.

Anyway, I'd suggest you use this as your last resort.

## Share your thoughts on this topic

My goal this semester is to make cards as soon as I learn a new topic because it was very stressful cramming so much material in just a few days. Of course I had studied during the semester, but I only started using anki weeks before the exams. 

Thanks for reading this guys! I'd really like to hear whether you found this helpful and what are your own methods for studying mathematics. Be sure to leave a comment bellow! Or join the discussion on [reddit](https://www.reddit.com/r/math/comments/exjpfo/international_student_surviving_in_a_russian/).

Also, you can check out my deck [here](https://ankiweb.net/shared/info/1561635088). I can't promise everything is perfect, as I said, I had very little time to make this, but it can serve as guide to making your own cards. Please let me know if you find any mistakes!
