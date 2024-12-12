(function() {
  'use strict';

  var plugins = ["o.js", "rezka.js"];
  var plugins_version = '?v=081224-4';

  var host = 'https://bwa.to';
  var framework = 'https://0f7214fe.bwa.pages.dev:8443';
  var framework_standby = 'http://185.245.106.167';

  function putRC() {
    Lampa.Utils.putScriptAsync(["http://default.rc.bwa.to/online.js"], function() {});
  }

  if (typeof WebAssembly == 'undefined') {
    putRC();
  } else if (Lampa.Platform.is('android') || Lampa.Platform.is('browser')) {
    var net = new Lampa.Reguest();
    net.timeout(2000);

    var standby = function() {
      framework = framework_standby;
      start();
    }

    net.native(framework + '/blazor.boot.json', function(j) {
      try {
        j.resources.assembly['JinEnergy.wasm'];
        start();
      } catch (e) {
        standby();
      }
    }, function(error) {
      standby();
    });
  } else {
    putRC();
  }

  function start() {
    function putplugins() {
      Lampa.Utils.putScriptAsync(plugins.filter(function(u) {
        return (!window.bwajs_plugin && u == 'o.js') || (!window['plugin_sisi_pwa_ready'] && u == 'rezka.js');
      }).map(function(u) {
        console.log('BWA', host + '/plugins/' + u + plugins_version);
        return host + '/plugins/' + u + plugins_version;
      }), function() {});
    }

    if (typeof WebAssembly === "object" && typeof WebAssembly.instantiate === "function") {  
      if (window.blazor_load == undefined) {
        setTimeout(function(){
          if (!window.blazor_init) putRC();
        }, 1000 * 10);

        window.blazor_load = true;
        var s = document.createElement('script');
        s.onload = function() {
          if (typeof Blazor == 'undefined') {
            console.log('BWA', 'Blazor undefined');
            return;
          }

          try {
            Blazor.start({
              loadBootResource: function(type, name, defaultUri, integrity) {
                return framework + '/' + name;
              }
            }).then(function() {
              console.log('BWA', 'load complete');
              var net = new Lampa.Reguest();
              window.httpReq = function(url, post, params) {
                return new Promise(function(resolve, reject) {
                  net["native"](url, function(result) {
                    if (_typeof(result) == 'object') resolve(JSON.stringify(result));
                    else resolve(result);
                  }, reject, post, params);
                });
              };

              var check = function check(good) {
                try {
                  fetchRezkaContent('movies', function(parsedData) {
                    console.log('Rezka Content:', parsedData);
                  });

                  var type = Lampa.Platform.is('android') ? 'apk' : good ? 'cors' : 'web';
                  var conf = host + '/settings/' + type + '.json';
                  DotNet.invokeMethodAsync("JinEnergy", 'oninit', type, conf);
                } catch (e) {
                  console.log('BWA', e);
                  putRC();
                }
              };

              if (Lampa.Platform.is('android') || Lampa.Platform.is('tizen')) check(true);
              else {
                net.silent('https://github.com/', function() {
                  check(true);
                }, function() {
                  check(false);
                }, false, { dataType: 'text' });
              }
            })["catch"](function(e) {
              console.log('BWA', e);
              putRC();
            });
          } catch (e) {
            console.log('BWA', e);
            putRC();
          }
        };

        s.setAttribute('autostart', 'false');
        s.setAttribute('src', framework + '/blazor.webassembly.js');
        document.body.appendChild(s);
      } else if (window.blazor_init) {
        putplugins();
      }
    } else {
      putRC();
    }
  }

  // Функция для работы с Rezka
  function fetchRezkaContent(type, callback) {
    var url = 'https://rezka-ua.tv' + (type === 'movies' ? '/movies/' : '/series/');
    var net = new Lampa.Reguest();
    net.native(url, function(data) {
      var parsedData = parseRezkaHTML(data);
      callback(parsedData);
    }, function(error) {
      console.error("Error fetching Rezka content:", error);
    });
  }

  function parseRezkaHTML(html) {
    var data = [];
    // Пример парсинга: используйте регулярные выражения или DOMParser для извлечения данных
    var parser = new DOMParser();
    var doc = parser.parseFromString(html, 'text/html');
    var items = doc.querySelectorAll('.b-content__inline_item');
    items.forEach(function(item) {
      var title = item.querySelector('.b-content__inline_item-link').textContent;
      var url = item.querySelector('.b-content__inline_item-link').href;
      data.push({ title: title, url: url });
    });
    return data;
  }
})();
