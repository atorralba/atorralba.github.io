---
layout: post
title: Blocking the AdBlock Blocker
---

I don't want to talk about politics here because I want this to be a tech blog, so I'll be quick. Let's just say that, given the political situation in my country right now, I wanted to watch an specific [interview][1] between a famous journalist and the current leader of the Catalonian Independence movement. The only problem was that the site that hosts the video of the interview doesn't allow its reproduction if some ad-blocking software is detected in the user's browser. That's extremely annoying since, if you disable it, you are not only exposed to the malware, viruses and COMMUNISM everyone knows is present in ads, but also they force you to watch like **three** 20~30 seconds long ads before even letting you start watching the video. I got *so* ~~annoyed~~ interested I decided to investigate how they detected the presence of ad-blockers and then ~~try to bypass it to watch the damn video in peace~~ check the robustness of the software in case it was insecure. Logical, right?

## Oops

So, if you go to the site, you will probably get blocked because their ToS say it only works for Spanish users, but I didn't try so I don't know. Anyway, if you go to the site and you are Spanish, you get a nice and big "Play" button in the middle of the screen, and when you, innocent and naive creature, click said button, your hopes get crushed by the evil masterminds of the ad-block-blockers industry. You receive the following warning and you can't access the video and all in life is wrong.

![Oops disable your adblocker or PAY US $$$](/images/oops.png)

It basically says "Oops you'll have to disable your ad-blocker or pay money to us if you want to see this". Right, sure, I feel you. *BUT*. How do *you* know that I'm blocking your ads? ARE YOU BY CHANCE SPYING ON ME? Well, then that means two things:

 1. My tinfoil hat didn't protect me as I expected.
 2. You summoned the *evil* forces of Javascript to invade my privacy and unlawfully **interact** with my browser extensions. Shame on you.

That flawless logic gives me 100% legit pretext to *"ethically"*® "audit"™ your code in search of, hum, noncompliances, yeah, whatever that means.

## The amazing land of obfuscated Javascript

So, with the almighty **Firefox Developer Tools**, we can search for the id of the ~~annoying~~ warning that blocks the video reproduction. After 30 *complete* seconds, we find that `mensaje-adblock` is being used in a script called `player.bundle.js`, and oh man it's FULL of obfuscated GARBAGE. There are references everywhere to open source tools like [this one][2] or [this one][3], so I imagine a more sophisticated professional who was *totally not like me* would investigate said tools and try to understand what the obfuscated code is doing. But since I'm a total script-kiddie, I just *furiously* **copy-paste** all the code into a [tool][4] that better engineers than me offered to the world in a complete display of love for humanity. This is what I get:

~~~~javascript
function(e) {
    function t(n) {
        if (o[n]) return o[n].exports;
        var c = o[n] = {
            exports: {},
            id: n,
            loaded: !1
        };
        return e[n].call(c.exports, c, c.exports, t), c.loaded = !0, c.exports
    }
    var o = {};
    return t.m = e, t.c = o, t.p = "dist/", t(0)
}([function(e, t, o) {
    "use strict";

    function n(e) {
        return e && e.__esModule ? e : {
            "default": e
        }
    }
    o(1);
    var c = o(5),
        i = n(c),
        r = void 0,
        a = function(e) {
            function t() {
                if (!r) {
                    r = p.addChild(p.createVjsComponent("Component")), r.el_.className = "vjs-detect-overlay", r.el_.innerHTML = d.template();
                    var e = $(p.el());
                    $(".vjs-detect-close", e).on("click touchstart", c), $(".vjs-detect-reload", e).on("click touchstart", a), $(".vjs-detect-button", e).on("click touchstart", d.onButtonClick)
                }
            }
            function o() {
                $(r.el()).remove()
            }
            function n() {
                d.onDetect && d.onDetect(), t(), p.paused() ? p.one("play", function() {
                    p.pause()
                }) : p.pause()
            }
            function c() {
                o(), d.rememberChoice && u(), p.play()
            }
            function a() {
                window.location.reload()
            }
            function u() {
                var e = d.cookieName,
                    t = d.cookieExpiration;
                document.cookie = e + "=true; expires=" + t.toGMTString() + "; path=/;"
            }
            function s() {
                var e = d.cookieName;
                return d.rememberChoice && new RegExp(e + "=").test(document.cookie)
            }
            if (e.detect) {
                var l = (new Date).getTime(),
                    d = Object.assign({
                        rememberChoice: !0,
                        cookieName: "skip_adb",
                        cookieExpiration: new Date(l + 12096e5),
                        template: i["default"],
                        onButtonClick: function() {}
                    }, e),
                    p = this;
                d.rememberChoice && !s() && d.detect().then(function(e) {
                    return e && n()
                })
            }
        };
    videojs.plugin("adblockDetectPlugin", a)
}, function(e, t) {}, , , , function(module, exports) {
    module.exports = function(obj) {
        obj || (obj = {});
        var __t, __p = "";
        with(obj) __p += '<p>Desactiva AdBlock para ver el vídeo</p>\r\n<button class="vjs-detect-reload">Ya lo he desactivado</button>\r\n<button class="vjs-detect-close">Continuar</button>\r\n';
        return __p
    }
  }
);
~~~~

Okay, I'll stop pretending I understand *any of that* anyway and focus on the single line that catches my otherwise shamefully short attention span.

~~~~javascript
videojs.plugin("adblockDetectPlugin", a)
~~~~

Now, if we check the definition of function `a`, we see it contains the definition of a handful of other functions with original names like `t`, `o` and of course our little friend `n`. Now, we are totally not in the mood to try to reverse-engineering *all that* because ~~we want to see our damn video~~ reasons. So, let's make an **educated** guess and say that `a` is the magic function that decides what happens when that *evil and illegal* `videojs.plugin` discovers we are running an ad-blocker. And it decides it won't play the video, apparently.

## "Noncompliance"

I decide it's worth a shot to try and call again to `videojs.plugin` but this time passing as second argument a function that does, like, *absolutely nothing*. Like telling this guy: "when you detect this good Samaritan is running any kind of ad-blocker, you can like *totally* chill, are we good?". The rocket-science, over-the-top code I *engineered* is the following:

~~~~javascript
videojs.plugin("adblockDetectPlugin", function(){});
~~~~

And, guess what. It *works*. The video plays smoothly, 100% *ad free*, with our ad-blocker guy freely running up there, like God intended him to be. Now, **of course**, as an "ethical"℠ guy, I *totally* stopped the video right there and reloaded the browser to enjoy my minute-and-a-half of delicious ads before finishing watching the video because I'm a good guy and don't want problems with law enforcement and I would **never** commit ~*crimes*~.

Oh, there's one more thing. Since I actually never developed any kind of browser extension, I thought this was an excuse as good as any other to get started and write my own "Hello, world!" Firefox extension that automates the *absolutely painful* process of pressing F12 and copy-pasting that one line of code into the Javascript console so anyone can "_ethically_" verify this "noncompliance" that I "found". Sorry I can't stop using quotation marks, it's a serious problem and I'm fighting it with all I've got but there's still a long way to complete recovery. Anyway, here's the [code][5].

## Conclusion

I decided to write this blog post because I spent a couple hours this Sunday evening playing with Javascript and didn't want to leave it in a simple "wow, I almost by pure luck bypassed this simple ad-block check". So here's this, in a shameful attempt of being more "fun" and less boring. I'm *slightly* worried that cops may come knocking to my door the second I upload this, because I may become responsible of the loss of MILLIONS in revenue if this gets popular and people starts AVOIDING ads like total criminals. So, *LEGAL DISCLAIMER* the tool is only for "demonstration" *scientific* **learning** _responsibility-discharging-word_ purposes and I cannot control what others use my innocent line of code for.

On the other hand, I'm totally voting the next Sunday in an **ILLEGAL** Independence Referendum organized in my region, so maybe I'll end up in jail anyway. Wish me luck!

[1]: http://www.atresplayer.com/television/programas/salvados/temporada-13/capitulo-1-una-hora-puigdemont_2017092100859.html
[2]: https://github.com/cladera/videojs-cuepoints
[3]: https://github.com/lancedikson/bowser
[4]: http://deobfuscatejavascript.com
[5]: https://github.com/atorralba/atresplayer-adblockblocker-blocker
