---
layout: post
title: "Получаем Premium на busuu.com бесплатно"
date: "2012-05-21 17:42:20"
comments: true
categories: busuu hack premium бесплатно
---

Недавно я начал пользоваться сервисом [busuu.com](http://www.busuu.com). Очень классная вещь, но вот беда: спустя неделю часть функционала перестала работать, предлагая купить Premium аккаунт за 69,99 EUR.

Одна из полезностей которые пропали, была возможность прослушивания ключевых фраз во время изучения новых слов. Кнопка, при нажатие на которую должна произносится фраза, превратилась в обычную ссылку на страницу, которая предлагает купить Premium.

![bussu play button](http://i1078.photobucket.com/albums/w484/greyblake/play_button.png)

Мне стало интересно, а можно ли это обойти? Оказалось, что можно! Файловый сервер, который отдаёт аудио-файлы никак не проверяет права пользователя, а сами ссылки на файлы можно прямо получить из ответов сервера на AJAX-запросы, которые отсылаются при нажатии на следующее/предыдущее слово.

После полутора часов изучения и дебага родился скрипт, приведенный в конце статьи, который делает то, что мне нужно:

* Возвращает старую кнопку, вместо ссылки
* Переопределяет callback AJAX-запроса так, чтобы он получал из ответа ссылку на аудио-файл.

##Как этим воспользоваться?

Предполагаю, что статью будут читать и нетехнические люди, поэтому опишу в деталях, как использовать эту уязвимость в своё благо.

### Установка Firebug

Firebug - это расширение для Firefox, которым обычно пользуются веб-разработчики. Вам придётся его установить.
Сделать это можно по [этой ссылке](http://getfirebug.com/), нажав на кнопку "Install Firebug". После чего броузер должен перезагрузиться.


### Основной трюк

Зайдите на busuu.com на страницу изучения слов. Откройте Firebug.
В моём случае это делается через меню: Tools -> Web Developer -> Firebug -> открыть Firebug. Возможно в разных сборках Firefox, это может быть по-разному.

Откройте консоль и полностью скопируйте в неё Javascript код, приведенный в конце этой статьи, как показано на рисунке. После чего нажмите кнопку "Выполнить".

![firebug](http://i1078.photobucket.com/albums/w484/greyblake/firebug-1.png)

### Изучайте ваш любимый язык

Дело сделано, вы можете закрыть Firebug. Теперь, кликнув на следующее слово у вас будет возможность прослушать ключевую фразу. Всё это будет работать до тех пор, пока вы не перезагрузите страницу.

## Если у вас не Firefox

В большинстве других броузеров есть подобные инструменты для отладки, которые позволят вам проделать такой же трюк. Например, в броузере Google Chrome он встроенный, и открывается при нажатии клавиш Ctrl+Shift+J. В видео ниже показано, как именно это делается.

<iframe width="420" height="315" src="http://www.youtube.com/embed/cVn7IkkQFBU" frameborder="0" allowfullscreen></iframe>


##Javascript код

```javascript put it into Firebug console


// Define languages
hack_native_lang = $('body').attr('lang')
var hack_matched = location.pathname.match(/\/(enc|ar|zh|fr|de|it|ja|pl|pt|ru|es|tr)\//)
hack_learning_lang = hack_matched[1]

// Replace link to subsribe with real play button
var hack_player_div = '<div id="player2" class="busuu-player"></div>'
$("#kp-audio").html(hack_player_div)

// Update onClick event
$("#player2").click(function(){
   if($(this).hasClass("busuu-player-pause")){
     if(!$("#player").data("jPlayer").status.paused){
       $(this).removeClass("busuu-player-pause");
       $("#player").jPlayer("pause");
     }
     else {
       $("#player").jPlayer("play");
     }
   }
   else{
     $("#player").jPlayer("setMedia", {mp3:kp_audio}).jPlayer("play");
     $(this).addClass("busuu-player-pause");
   }
});

// Redefine function getEnt() to make it set global variable kp_audio with link to mp3 file
function getEnt(pl,entid) {
   p_entid = (entities[index -1] > 0) ? entities[index -1] : 0;
   n_entid = (entities[index +1] > 0) ? entities[index +1] : 0;
   if (entid == 14069255 || entid == 7312 || entid == 1305 || entid == 3279 ||
       entid == 2310 || entid == 81017 || entid == 1450 || entid == 30872){
     $("#ge").show("drop");
   }
   else {
    $("#ge").hide();
   }
   $.ajax({
       type: "GET",
       url: "/js/busuuajax/entity/"+entid+"/"+hack_learning_lang+"/"+hack_native_lang+"/" + p_entid + "/" + n_entid,
       cache:true,
       dataType: "xml",
       success: function(xml) {
              $("#report_link").attr("href", "/js/busuuajax/user_report_entity/"+hack_learning_lang+"/1_1_4/"+entid+"?KeepThis=true&TB_iframe=true&height=270&width=300");
              $("#star").attr("src","http://static3.bscdn.net/sites/all/themes/busuunew/images/star_off.png");
              already_selected = $(xml).find("flagged").text() == "1" ? true : false;
              if (already_selected )$("#star").attr("src","http://static1.bscdn.net/sites/all/themes/busuunew/images/star_on.png");
              image = "<img style='display:none;' src='"+$(xml).find("image").text()+"' />";
              prev_image=$(xml).find("prev_image").text();
              next_image=$(xml).find("next_image").text();
              audio = $(xml).find("audio").text();
              word1 = $(xml).find("value1").text();
              word2 = $(xml).find("value2").text();
              phrase1 =  $(xml).find("keyphrase1").text();
              phrase2= $(xml).find("keyphrase2").text();
              phonetics = $(xml).find("entity_phonetics").text();
              if(phonetics.length > 0){
                entity_phonetics = "<br><div dir=\"ltr\" lang=\"enc\" class=\"phonetics\">";
                entity_phonetics += phonetics;
                entity_phonetics += "</div>";
              }
              phonetics = $(xml).find("kp_phonetics").text();
              if(phonetics.length > 0){
                kp_phonetics = "<br><div dir=\"ltr\" lang=\"enc\" class=\"phonetics\">";
                kp_phonetics +=  phonetics;
                kp_phonetics += "</div>";
              }
              kp_audio= $(xml).find("kp_audio").text();

              if(navigator.platform.indexOf("iPad") != -1){
                  $("#player").jPlayer("setMedia", {mp3: audio});
              }
              else{
                $("#player").jPlayer("setMedia", {mp3: audio}).jPlayer("play");
                $("#player1").addClass("busuu-player-pause");
              }


              if (index > 0 ){
                $("#left").html("<img style='cursor:pointer;' src='"+ prev_image + "' width=100  height=66>");
                $("#left_arrow").show();
                $("#left_border").show();
                $("#left").show();
              }
                if (index < entities.length -1 ){
                $("#right").html("<img style='cursor:pointer;'  src='"+ next_image + "' width=100  height=66>");
                $("#right").show();
                $("#right_arrow").show();
                $("#right_border").show();
              }
              $(".lu-learn-phraseA").html(word1 + entity_phonetics);
              $(".lu-learn-phraseB").html(word2);
             kp_audio_html='<div id="player2" class="busuu-player"></div>';
              if (!$(xml).find("keyphrase1").text()=="") {
                $(".table-lu-key-phrases").show();
                if (!kp_audio=="") {
                  $("#kp1").html(phrase1 + kp_phonetics);
                  $("#kp_divider").show();
                  $("#kp2").html(phrase2);
                  $("#kp-audio").html(kp_audio_html);
                  activatePlayer2();
                }
                else {
                  $("#kp1").html(phrase1 + kp_phonetics);
                  $("#kp_divider").show();
                  $("#kp2").html(phrase2);
                }
                  $(".table-lu-key-phrases").show();
              }
              else {
                $("#kp_divider").hide();
                $(".table-lu-key-phrases").hide();
              }
              $(".lu-review-ratio").html((index +1) +"/" + entities.length);

                $("#middle").html(image);
                $("#middle").children().fadeIn(300);
           }
      });
}
```
