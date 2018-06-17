It's been about a month and a half since I released [BYTEPATH](http://store.steampowered.com/app/760330/BYTEPATH/) and I think it's been long enough that writing a postmortem makes sense. I've already written a more technical and programming oriented postmortem-ish article [here](http://ssygen.com/posts/engine-2018) so go check it if you're interested in that. This article will focus more on anything that isn't programming related.

<br>

## Context 

This game came about because I had failed for about 5 years on 3 other projects prior to this one, and I decided that I had enough and that I'd just finish and release a game no matter what. I picked a relatively simple gameplay loop to make by copying [Bit Blaster XL's](http://store.steampowered.com/app/433950/Bit_Blaster_XL/). Then I tried to pick something else that would keep people playing the game for longer than if it was just like Bit Blaster XL, and the best way I know of doing that is just adding lots of variety to it, so I decided to copy the idea behind [Path of Exile's passive skill tree](https://www.pathofexile.com/passive-skill-tree/3.2.0/). This meant that while the gameplay was simple, the surrounding systems had enough complexity that I could grow and learn something from a technical perspective, which is also good since generally I don't want to be working on projects that I learn nothing from. 

<p align="center">
<img src="https://i.imgur.com/UI2iZgf.jpg">
</p>

On top of making the game I also wrote [a tutorial](https://ssygen.itch.io/bytepath-tutorial) on how to make it from scratch. I wrote this tutorial both because I wanted to see if I actually enjoyed writing tutorials in a thorough manner, and to also check if the "tutorial market" was a viable way of making money on top of just making games. I think it's well known that there are many young people who are interested in making games and it would follow that those people would be willing to spend money on material aimed at teaching them how to make a game. And so it's worth it to check if there are enough of them to make writing said material worth it.

<br>

## Expectations

The game itself was made entirely with programmer art and a few shaders here and there, so my expectations for how it would do were low. My goal going into this project was to just make the $100 I paid to get on Steam back, which meant that at $2 a copy, I'd have to sell 500 copies, since you need to make at least $1000 to make the entry fee back. I think for someone's first game and for a game with no proper art that's a good goal.

<p align="center">
<img src="https://i.imgur.com/Kl82nw1.png">
</p>

As for the tutorial I was set on selling each copy (which contained the game's source code, answers to exercises and a Steam key to the game) at $25. My expectations were that I wouldn't really sell that many copies, like 10 at most, so making the price a lot higher like that made sense. The reason for this is that while the tutorial is very thorough and educational, the number of people who are interested in LÖVE and Lua would be pretty low, and the number of people in that pool that would be OK with paying $25 for source code would be even smaller.

As for marketing efforts, my plan was to just post the game and the tutorial on a few specific subreddits and on Hacker News. My posts have been "at the top" of a bunch of subreddits and of HN before and so I know that the number of views that these sites generate can get pretty crazy. And my thought process was that if I could just repeat that again for this game it would give it a really good visibility boost. I knew the tutorial itself would get to the top of [/r/programming](https://www.reddit.com/r/programming/), [/r/gamedev](https://www.reddit.com/r/gamedev/) and [HN](https://news.ycombinator.com/news) without many problems, because it's just a lot of content and a lot of good content, and generally it's hard for people to ignore and not upvote stuff like that. As for the game itself I wasn't sure if I would make it to the top of any of my target subreddits because the game just didn't look good enough.

<br>

## Financial Results

The combined amount of money made on both the game and the tutorial, after all cuts and taxes, was about $3k. I live in Brazil, so after currency conversion this amounts to ~R$10k. This was well beyond my initial expectations so in this sense the game was a huge success. If we were to look at this from a "can I do this for a living" perspective then it wasn't a success though. The game itself took about 4 months to make, and the tutorial took about the same amount of time. These 8 months were spread unevenly along 1 year where in between I had breaks where I worked on other projects or didn't work on anything. If we take the amount of time it took for this project to be anywhere between 8 and 12 months then I made either slightly below local minimum wage or slightly above it, which isn't enough to do it for a living.

However, I think it's unreasonble to expect your first game to be a massive success or anything like that, and it's also unreasonable to expect that of a game with no art. At the end of the day this game just wasn't good enough and the market responded appropriately to its quality. One of the big mistakes I see other indie developers making is not being realistic about their experience and the quality of their games, and if you can't do that then you're setting yourself up for disappointment.

As for the tutorial, I didn't enjoy writing it as much as I thought I would, and it took the same amount of time to make as the actual game. So even though the tutorial sales also exceed my expectations by a lot, in the future I'm not going to write more tutorials like this. They just take too much time and effort for not enough gain. If I were using a more popular engine like Unity then the pool of people who would be interested would be higher and maybe it would be worth it, but since I'm not then that's that.

<br>

## Marketing Results

The most important lessons I learned from making this game were technical ones, but the second most important were marketing related. A lot of the "success" that I was able to have was due to my marketing efforts so I think it's a good idea to expand on those.

<br>

### Design with marketing purpose #1

The main piece of the game's design that I settled on that would also serve as its main marketing tool was the huge skill tree. I still remember when I first saw Path of Exile's skill tree and the feeling it gave me and it seems that most people when looking at it for the first time have that same feeling:

* [https://clips.twitch.tv/OriginalLittleGrasshopperLitFam](https://clips.twitch.tv/OriginalLittleGrasshopperLitFam)

* [https://clips.twitch.tv/SpikyAltruisticBobaTakeNRG](https://clips.twitch.tv/SpikyAltruisticBobaTakeNRG)

* [https://clips.twitch.tv/CloudyCrispyBillSaltBae](https://clips.twitch.tv/CloudyCrispyBillSaltBae)

And so my idea was to make my own version of that. The results of this weren't as good as I'd hoped for because my skill tree didn't look good, I think. While some people were impressed by it and got the game solely because of this, my overall conversion rate on the Steam store was pretty low I think, which means that having a picture of the skill tree as the first picture that was displayed when people hovered over the game didn't work that well. 

<p align="center">
<img src="https://i.imgur.com/Mpphxcq.png">
</p>

<br>

### Design with marketing purpose #2

The second way in which I designed the game with marketing in mind was by making it a console/terminal/hacking type of game. Because the game used programmer art I had very little options in terms of how I would present the game thematically, and so this one just fit like a glove. There are also a number of people who like games that contain terminals in them, and so this matched it perfectly. Additionally, I also made sure to port the game to Linux, to further increase my chances of capturing the types of people who are interested in terminal games in general. 

<p align="center">
<img src="https://i.imgur.com/fDQE27P.png">
</p>

The Linux idea turned out to be not so great because they amounted to like 5% of my sales. Not a lot, really. But there were a number of people who bought the game and mentioned that they bought it alone for the console/terminal part of it, and not because of the skill tree. Which means that as an additional marketing tool it sort of worked.

<br>

### Tutorial marketing

I managed to get my tutorial on the top of Hacker News for half a day with [this thread](https://news.ycombinator.com/item?id=16404230), on the top of /r/programming for a day with [this thread](https://www.reddit.com/r/programming/comments/7xip6o/i_wrote_a_tutorial_on_how_to_make_a_complete_game/) and on the top of /r/gamedev with [this thread](https://www.reddit.com/r/gamedev/comments/7ximye/i_wrote_a_tutorial_on_how_to_make_a_complete_game/). I posted these about a week before the game was going to be released because I thought it would be a good idea to generate some visibility for the game before people have the opportunity to buy it in the first place. This decision was OK I think but I wonder how things would have gone if I released the tutorial and the game on the same day. In any case, these threads generated about 25k views on my GitHub blog over the next few days when they were released:

<p align="center">
<img src="https://i.imgur.com/URJxGbF.png">
</p>

And this also generated a similar looking graph on itch.io in terms of tutorial sales:

<p align="center">
<img src="https://i.imgur.com/BgYvnqB.png">
</p>

Overall the response to the tutorial in terms of attention it got was expected, the number of purchases surprised me though, since I thought they would be lower. One of the strategies I used to increase the chances that the thread I created about this on HN would go to the top was to post it at low traffic times. I think on HN I posted the thread on a Saturday night, and so that meant that just a small amount of upvotes quickly made the thread rise up to the top and then it stayed on the front page for a while. On reddit's smaller subreddits doing this isn't necessary, especially on /r/programming and /r/gamedev, since those subs are small enough that good content naturally rises to the top no matter the time. But on bigger subreddits a strategy like this might also help.

<br>

### Game marketing

About a week after posting the tutorial I released the game and my main way of marketing the game would be through reddit as well. The primary subreddit that I wanted to target was [/r/pathofexile](https://www.reddit.com/r/pathofexile/). My game is inspired by it, it's a very big subreddit full of people who are interested in that game, so it just makes the most sense to do it there primarily. One of the obvious problems with this is that I'm posting about a game that isn't PoE on the PoE subreddit. I had no way of knowing if the moderators would be cool with this, or even if the developers of the game would be cool with this, since they're very active on the subreddit and browse it all the time. 

However, I did my best to increase the chances of it being OK by timing it correctly, I think. One of the things about PoE is that every 3-4 months they release a new league to the game. The next league was going to come out on March 2nd, and what happens is that as the league goes people gradually stop playing and then 1-2 weeks before the next league starts they start getting hyped for it again. My idea was to release the game on Feburary 23rd, 1 week before the new league, to maximize on the number of people who are getting hyped about the new league but not actually playing PoE, but to also not intrude too much on the next league's actual release date, since the devs like to do a lot of hyping up themselves on the subreddit by releasing a lot of previews of what's coming (and they generally like to do this 1-2 weeks before the league releases).

I posted the game on the subreddit on [this thread](https://www.reddit.com/r/pathofexile/comments/7zu9vz/i_made_a_small_game_inspired_by_poes_passive/) and I used the same strategy that I mentioned above for HN. I posted it on a Friday night, at an hour of very low traffic. Another advantage of posting at this time is that the developers of the game are from New Zealand, which meant that they were probably well awake and checking the subreddit. This meant that if they weren't OK with this kind of shilling on their subreddit they would be able to do something about it quickly if they wanted to. 

After posting it immediately got a bunch of upvotes and shot up to the front of the subreddit. One of the developers of the game [commented on it rather quickly](https://www.reddit.com/r/pathofexile/comments/7zu9vz/i_made_a_small_game_inspired_by_poes_passive/duqti8t/) and then the main developer [also did](https://www.reddit.com/r/pathofexile/comments/7zu9vz/i_made_a_small_game_inspired_by_poes_passive/dur2mb5/). So at the end of the day they were OK with it! :D However, after about 1 hour of the thread being up it got removed by the moderators because they didn't allow for products to be linked on their subreddit like that. I messaged them and after a few minutes of internal discussion they reinstated the thread! And after that it stayed on the front page of the subreddit for about 1 day and generated around 40k views total.

I posted the game on a few other smaller subreddits as well and I tried posting it on HN but it didn't really get any traction there. At the end of the day the game was put on the radar and got a lot of attention because of the /r/pathofexile thread, so thanks to the developers and to the moderators for being OK with it ^^

<br>

### Further marketing

The only other marketing I did was to write a programming/technical postmortem on the game about 3 days after its release. I have many opinions that would be considered controversial by most people when it comes to a lot of subjects, so here the only notable thing I did was to write an article that had many of those controversial opinions because that would generate discussion around it and hopefully spread the article more. This somewhat worked well. I posted the article on [/r/programming](https://www.reddit.com/r/programming/comments/805xig/programming_lessons_learned_from_releasing_my/) and [/r/gamedev](https://www.reddit.com/r/gamedev/comments/80w52o/programming_lessons_learned_from_making_my_first/) as you can see it generated a lot of lively discussion. In terms of numbers this article managed to get about the same amount of views as when I first posted the tutorial, which was very good:

<p align="center">
<img src="https://i.imgur.com/CCJSPme.png">
</p>

This article took me less than a day to write and for the amount of time it took it was a very good amount of visibility. Overall, all these marketing efforts amounted to about 40% of the visits on my game's Steam page, which is higher than I would have expected. Steam itself generated TONS of views on my game, but because I didn't do a good job at converting those people it didn't generate as many visits as one would expect.

And other than that, I didn't contact any streamers, youtubers or journalists, because I didn't believe this game would get any traction with any of those people. Again, it's a game that doesn't look good at all and it would have been mostly a waste of time to try.

<br>

## Improvements for the Future

Despite a lot of my efforts doing well, there are a lot of things that I didn't do, either because I had no time, or because I just didn't think this game deserved the amount of effort it would take. I still think those things are important though so I might as well list them out too.

<br>

### Trailer

BYTEPATH's trailer turned out OK I guess but nothing special:

<p align="center">
<a href="https://www.youtube.com/watch?v=vRC1F1BSW7E"><img src="https://i.imgur.com/ar3WwlJ.jpg"></a>
</p>

And it turned out this way because I'm not too experienced with making trailers yet but also I didn't have the tools I wanted available in hand. However, I deeply believe that the trailer is the most important marketing asset you have for your game, and that it actually should take precedence over making the game. One of the things that I wanna do in the future is that once I'm done with prototyping some game and I decide that I'm going to make it for real, I will first make a trailer for it, without the game actually being done at all. 

This serves the main purpose of figuring out how to make the game marketable. You can have a really cool game prototype that plays well, but it means nothing if it isn't a marketable idea. Forcing and releasing a trailer before committing to making the full game will serve as a very good test of its marketability: it will either fade into obscurity or get some attention. If it fades into obscurity then you know that you need to change the way you present it or change games altogether, and if it gets some attention then you know you're on the right path, and you also get some community building going on.

<br>

### Game design with marketing in mind

This goes in accordance with the previous point, but the game should be thought up with how marketable it is in mind from the very beginning. For BYTEPATH this took the shape of the skill tree and of the console/terminal look, but it also took the form of having a target "place" where I would post the game on release too, like /r/pathofexile. I think this is also something really important that people don't mention or think about often.

My next game, [Frogfaller](https://twitter.com/SSYGEN/status/826903247961079808), has this problem. When I was first thinking it up I wasn't taking this into account that much, so now I have a design in my hands that I just don't know how to market yet. I can't think of a single place on the Internet where this game is a perfect fit for, like /r/pathofexile was for BYTEPATH, and I think that this is very bad because it just makes marketing it harder. On the other hand, the artist I'm working with on it makes super good looking art so that also makes it easier to market it, but ideally you want both good art and a good and well defined target audience that can be reached easily.

For instance, the game I'm planning to make after Frogfaller will be a clone of [Recettear](http://store.steampowered.com/app/70400/Recettear_An_Item_Shops_Tale/). It will have the anime style and similar gameplay, and I can think of like 100 places on the Internet where if I said something like "I'm making an anime game like Recettear" and I posted a good looking screenshot with proper anime art and all that, it would be very very well received. So again, it's simply a matter of designing a game with a well defined group of people in mind. If I can't think up of a few places where I'll post the game when it releases then I would probably rethink the game entirely.

<br>

### Streamable game

I've been watching a lot of Twitch streams lately as well and streamers in general can be a very good source of sales. But for streamers to play a game it has to be "streamable". I haven't managed to logically pin down what makes a game streamable yet entirely, but if you give me a list of 10 games and have me watch 30 minutes of gameplay of each, I'll be able to tell you which ones are streamable and which ones are not. 

In general people have a sense that roguelites for instance are streamable, because if you die you can just start another run and play what essentially amounts to a different game. But I think there's a more general idea behind it that has to do with unexpected things happening. Horror games, for instance, seem to be very streamable for this reason. A game like Path of Exile on the other hand doesn't have that going for it at all. A good recent example is [Getting Over It](http://store.steampowered.com/app/240720/Getting_Over_It_with_Bennett_Foddy/), which has a gameplay loop that just consistently generates "streamable moments" in the form of progression loss:

* [https://clips.twitch.tv/UnsightlyDifferentAlligatorVoteNay](https://clips.twitch.tv/UnsightlyDifferentAlligatorVoteNay)

* [https://clips.twitch.tv/CloudyImpossibleJackalMrDestructoid](https://clips.twitch.tv/CloudyImpossibleJackalMrDestructoid)

As well as general taunting and scares:

* [https://clips.twitch.tv/DullAntediluvianAxeBlargNaut](https://clips.twitch.tv/DullAntediluvianAxeBlargNaut)

* [https://clips.twitch.tv/IncredulousDeadGarageCoolStoryBob](https://clips.twitch.tv/IncredulousDeadGarageCoolStoryBob)

I feel like you want a game that generates moments that brings the streamer and chat together and that those moments more easily happen when something unexpected happens. But they can also happen if you make a game with Twitch chat integration, for instance. In those cases the chat can vote of what happens in the game and they can decide to fuck the streamer over or help him. Another way of looking at this is that you want to make the streamer's job (entertaining thousands of people) easier, so you need to do a lot of the heavy lifting in your game and place the streamer in positions where he can easily just react to something that happened in the game.

<br>

### Art

Finally, with making a marketable and streamable game in general comes the idea of making the game look good. This is another area where I think lots of people don't focus on enough I think. However before I go on I should say that I'm not an artist so what I'm about to say next might be wrong or misguided, but it's what I currently believe to be true. 

A lot of artists that I follow seem to have the problem that they can't "compose a screen" properly. I don't know if there's an actual art term for "composing a screen", but it basically means putting the pieces of the game together on the screen in a way that looks good. There are many many many artists who I see can draw individual assets really well, but they just can't put those assets together in a way that looks pleasing. And this is something that sinks A LOT of indie games that would otherwise do well, in my opinion. It's by far one of the most important things I look for in an artist if I'm going to make something with one. Here's one example of a game that fails at this in my opinion:

<p align="center">
<a href="https://www.youtube.com/watch?v=M0N3sEuRJ_I"><img src="https://i.imgur.com/bZztaRu.jpg"></a>
</p>

Tangledeep is an example where the assets look good but for one reason or another (perhaps noisy background assets) it just doesn't come together as a good looking screen. Obviously a lot of this is subjective but it's something that I pay attention to a lot and that I think more people should pay attention to as well. I'm sure that artists talk about this and they have proper terms for this idea and it's actually something that people study (hopefully), so it's not like people can't get better at it.

<br>

### Marketing in general

I talked a lot about marketing in this post, and in general I think it should be my goal to design a game that markets itself so that marketing it is easier, but that I should also focus my marketing efforts on places that I understand. The sites I use right now that can be used for marketing purposes are in order of use frequency: twitch, 4chan, reddit and YouTube. Along with that there are a bunch of Discord servers I frequent that can be used for marketing as well. 

With that in mind, I think it's reasonable to focus my marketing efforts on those 5 places that I frequently use and that I understand well, while avoiding the ones that I don't use at all. For instance, I don't read and have never read any gaming journalism site. While those sites can generate a lot of views for me, it just seems like a waste of time to cold e-mail hundreds of journalists who will end up ignoring any of my games when I don't even use their service. 

It makes much more sense to market on a service that I actually enjoy using because will be advantages that come with understanding how it works that will make my job easier. For instance, with Twitch generally you can just donate a small amount of money to a streamer and a robot will read your message out loud and the streamer will respond to it. If a streamer is ever looking for a game to play live, suggesting your game is something that you might do that could be very very effective. The worst thing that could happen is that you would waste like $5. The same goes for reddit, I use reddit all the time, which means that I understand how the rules of the site work and that I have a very old account that is unlikely to be confused for a random spammer.

Marketing in the places that you frequently use is something that can get you tons of advantages, and avoiding the places that you don't use can also save you time and effort that would have been better spent elsewhere. On top of that, "shaping" yourself to use the most popular sites and services is a useful skill to have. If you're someone who hates streaming you might want to not hate it so much and eventually even like it, because it's useful as an indie to be in touch with the gaming community as much as possible, since it will make your job a lot easier.
