/**
 * Estrae il DOMText node dagli elementi di una pagina HTML
 * per renderizzarli come audio tramite API Google
 * Utilizza un proxy server side per eliminare il problema del referer
 * 
 * @package SCREENREADER::plugins
 * @author JExtensions Store
 * @copyright (C) 2016 - JExtensions Store
 * @license GNU/GPLv2 http://www.gnu.org/licenses/gpl-2.0.html
 */
(function($) {
	$.frTTSEngine = function(configOptions) {
		/**
		 * Stato di attivazione del color changer
		 * @access private - getter closure
		 * @var Boolean
		 */
		var active;
		
		/**
		 * Stato della riproduzione play/pause/stop
		 * @access private - getter closure
		 * @var String
		 */
		var readerStatus;
		
		/**
		 * Reference al timeout del preloading
		 * @access private - getter closure
		 * @var String
		 */
		var preloadTimeout;

		/**
		 * Testo completo del nodo di testo selezionato nella pagina
		 * @access private
		 * @var String
		 */
		var fullInnerText;
		
		
		/**
		 * Indice di inizio dinamico
		 * @access private
		 * @var Int
		 */
		var startDynaIndex;
		
		/**
		 * Indice di fine dinamico
		 * @access private
		 * @var Int
		 */
		var endDynaIndex;
		
		/**
		 * Limite naturale di lunghezza chunk
		 * @access private
		 * @var Int
		 */
		var limit;
		 
		/**
		 * Lingua ottenuta tramite JFactory::getLanguage(); nel server domain
		 * @access private
		 * @var string
		 */
		var languageCode;
		  
		/**
		 * Object delle options dell'engine
		 * @access private
		 * @var Boolean
		 */
		var TTSConfigOptions;
		
		/**
		 * Chunks addizionali calcolati nel playing del primo chunk
		 * @access private
		 * @var Array
		 */
		var additionalChunks = new Array();
		
		/**
		 * ID current chunk sound in riproduzione
		 * @access private
		 * @var Array
		 */
		var currentSoundID = null;
		
		/**
		 * ID current chunk testuali no preload
		 * @access private
		 * @var Array
		 */
		var textChunksNoPreloadCounter = 0;
		  
		/**
		 * Ingloba la vera logica di riproduzione del TTS operando ricorsivamente
		 * su chunk di testo presi a partire dal fullInnerText in compliance
		 * all'API Google
		 * Una volta terminata la condizione ricorsiva esce chiamando la callback dei messaggi
		 * @access private
		 * @recursive
		 * @param String chunk
		 * @return Void
		 */
		function playChunk(chunk) { 
			// Se siamo nella prima call e il chunk non passato partiamo dal fullInnerText
			startDynaIndex = 0;
			endDynaIndex = findEndIndex(startDynaIndex + limit);
			firstChunk = fullInnerText.substring(startDynaIndex, endDynaIndex);
			
			// Se la lunghezza del firstChunk non e' nulla si procede alla lettura
			if(firstChunk.length) {
				 soundManager.createSound({
					 id: 'chunkSound',
					    url: TTSConfigOptions.baseURI + 'libraries/tts/' + TTSConfigOptions.readerEngine + '.php?lang=' + languageCode + '&engine_token=' + TTSConfigOptions.engineToken + '&token=' + TTSConfigOptions.token + '&text=' + encodeURIComponent(firstChunk),
					    autoLoad: false,
					    type: 'audio/mp3'
					  });
				 soundManager.play('chunkSound',{onfinish: function(){
					// All'evento onfinish si distrugge sempre
					 soundManager.destroySound('chunkSound'); 
					 // Check for additional preloaded chunks
					 if(additionalChunks.length) {
						 if(parseInt(TTSConfigOptions.preload)) {
							 recursivePlay(additionalChunks); 
						 } else {
							 recursiveCreatePlay(additionalChunks); 
						 }
					 } else {
						// Show user message
						showMessages('finished');
						return;
					 } 
				 }});
				 // Set current sound ID
				 currentSoundID = 'chunkSound';
				 soundManager.setVolume(currentSoundID, TTSConfigOptions.screenReaderVolume);
			} else {
				// Show user message
				showMessages('notext');
				return;
			} 
			
			// Se abbiamo ancora testo da leggere continuiamo ricorsivamente
			if (fullInnerText.substr(endDynaIndex) && parseInt(TTSConfigOptions.preload)) {
				 var counter = 0;
				 var extractSubString = function() {
					// Inversion of start
					 startDynaIndex = endDynaIndex;
					 endDynaIndex = findEndIndex(startDynaIndex + limit);
					 chunk = fullInnerText.substring(startDynaIndex, endDynaIndex);
					 var chunkSound = soundManager.createSound({
						 	id: 'chunkSound' + counter,
						    url: TTSConfigOptions.baseURI + 'libraries/tts/' + TTSConfigOptions.readerEngine + '.php?lang=' + languageCode + '&engine_token=' + TTSConfigOptions.engineToken + '&token=' + TTSConfigOptions.token + '&text=' + encodeURIComponent(chunk),
						    autoLoad: false,
						    type: 'audio/mp3'
						  });
					 chunkSound.load();
					 additionalChunks.push(chunkSound);
					 counter++;	 
					 
					 if(fullInnerText.substr(endDynaIndex)) {
						 preloadTimeout = setTimeout(function() {
							 if(readerStatus !== 'stop') {
								extractSubString();  
							 }
						 }, 3000);
					 }
				 }
				 // First call
				 preloadTimeout = setTimeout(function() {
					 if(readerStatus !== 'stop') {
						 extractSubString(); 
					 }
				 }, 3000);
			 } else if(fullInnerText.substr(endDynaIndex)) {
				 var counter = 0;
				 while(fullInnerText.substr(endDynaIndex)) {
					 // Inversion of start
					 startDynaIndex = endDynaIndex;
					 endDynaIndex = findEndIndex(startDynaIndex + limit);
					 chunk = fullInnerText.substring(startDynaIndex, endDynaIndex);
					 additionalChunks.push(chunk);
					 counter++;
				 }
			 }
		};
		
		/**
		 * Riproduce tutti i restanti chunks gia in preloading
		 * @access private
		 * @recursive
		 * @param String chunk
		 * @return Void
		 */
		function recursivePlay(soundsArray) {
			var elem = soundsArray.splice(0,1);
			if($.isArray(elem) && elem.length) {
				soundManager.play(elem[0]['id'], {onfinish: function(){
					 // All'evento onfinish si distrugge sempre
					 soundManager.destroySound(elem[0]['id']); 
					 recursivePlay(soundsArray);
				 }});
				// Set current sound ID
				currentSoundID = elem[0]['id'];
				soundManager.setVolume(currentSoundID, TTSConfigOptions.screenReaderVolume);
			 } else {
				// Show user message
				showMessages('finished');
				return;
			 }
		};
		
		/**
		 * Riproduce tutti i restanti chunks gia in preloading
		 * @access private
		 * @recursive
		 * @param String chunk
		 * @return Void
		 */
		function recursiveCreatePlay(soundsArray) {
			var elem = soundsArray.splice(0,1);
			var chunkID = 'chunkSound' + textChunksNoPreloadCounter;
			if($.isArray(elem) && elem.length) {
				var chunkSound = soundManager.createSound({
					 	id: chunkID ,
					    url: TTSConfigOptions.baseURI + 'libraries/tts/' + TTSConfigOptions.readerEngine + '.php?lang=' + languageCode + '&engine_token=' + TTSConfigOptions.engineToken + '&token=' + TTSConfigOptions.token + '&text=' + encodeURIComponent(elem[0]),
					    autoLoad: false,
					    type: 'audio/mp3'
					  });
				soundManager.play(chunkID, {onfinish: function(){
					 // All'evento onfinish si distrugge sempre
					 soundManager.destroySound(chunkID); 
					 recursiveCreatePlay(soundsArray);
				 }});
				// Set current sound ID
				currentSoundID = chunkID;
				soundManager.setVolume(currentSoundID, TTSConfigOptions.screenReaderVolume);
				textChunksNoPreloadCounter++;
			 } else {
				// Show user message
				showMessages('finished');
				return;
			 }
		};
		
		/**
		 * Distrugge tutti i chunks in play
		 * @access private
		 * @return Void
		 */
		function destroyPlay() {
			// Destroy di riproduzioni in corso
			soundManager.stopAll();
			soundManager.destroySound('chunkSound');
			$.each(soundManager.sounds, function(index, sound){
				soundManager.destroySound(sound.id);
			});
			additionalChunks = new Array();
			// Reset all current sound ID
			currentSoundID = null;	
		}
		
		/**
		 * Determina l'indice numerico terminale del chunk da leggere
		 * sulla base del successivo spazio rispetto al normale indice terminale
		 * @access private 
		 * @param Int naturalIndex
		 * @return Int L'indice determinato
		 */
		function findEndIndex(naturalIndex) {
			// Se il naturalIndex e gia uno spazio non serve compiere nessuna operazione
			if(fullInnerText.charAt(naturalIndex) == ' ') {
				return naturalIndex;
			}
			
			// Altrimenti si procede a spostarsi sul successivo spazio di interruzione parole 
			var nextChunkString = fullInnerText.substr(naturalIndex);
			 
			// Raggiunta la fine della stringa quindi nessuno spazio successivo e si restituisce la lunghezza rimanente come endDynaIndex
			if(nextChunkString.indexOf(' ') == -1) {
				var totalRemaining = nextChunkString.length + naturalIndex;
				return totalRemaining;
			} else {
				var nextSpaceIndex = nextChunkString.indexOf(' ') + naturalIndex;
				return nextSpaceIndex;
			} 
		};
  
		/**
		 * Aggiunge gli steps del volume control
		 * @access private 
		 * @return String Il testo trovato
		 */
		function addVolumeSteps() {
			var volumeContainer = $('#volume_plugin');
			var volumeLimit = TTSConfigOptions.screenReaderVolume / 20;
			// Numero di steps
			var numSteps = 5
			for (var i=0; i < numSteps; i++) {
				var activeLimitClass = !!(i < volumeLimit) ? ' active' : '';
				$(volumeContainer).append('<div id="volume_step' + (i+1) + '" class="volume_step' + activeLimitClass + '" data-value="' + parseInt((i+1)*20) + '" accesskey="' + (i+1) + '"></div>');
			}
		}
		
		/**
		 * Determina il testo selezionato sulla pagina da dare in pasto allo screen reader
		 * @access private 
		 * @return String Il testo trovato
		 */
		function findTextToRead() {
		    var text = "";
		    
		    // Mode 1 - text selection WIN over other read mode
		    if (window.getSelection) {
		        text = window.getSelection().toString();
		    } else if (document.selection && document.selection.type != "Control") {
		        text = document.selection.createRange().text;
		    }
		    
		    // Mode 2 - read by page parts FALLBACK if enabled and not selected text
		    if(!!parseInt(TTSConfigOptions.readPage) && !text) {
		    	if(!!TTSConfigOptions.readChildNodes) {
		    		// Exclude scripts from the HTML inline code if any
		    		if(parseInt(TTSConfigOptions.excludeScripts)) {
		    			var clearedTree = $(TTSConfigOptions.mainpageSelector)
		    			.clone()
		    			.find( "script" )
		    			.remove()
		    			.end()
		    			.filter(function(){
		    				if (this.nodeName && this.nodeName.toLowerCase() == 'script') {
		    					return false;
		    				}
		    				return true;
		    			});
		    			text = $.trim($(clearedTree).text()); 
		    		} else {
		    			text = $.trim($(TTSConfigOptions.mainpageSelector).text()); 
		    		}
		    		
		    		if(parseInt(TTSConfigOptions.readImages)) {
		    			var imagesText = '';
		    			var imagesNodes = $(clearedTree).find('img');
		    			$.each(imagesNodes, function(index, image){
		    				var thisImageText = $(image).attr(TTSConfigOptions.readImagesAttribute);
		    				if(thisImageText) {
		    					imagesText += thisImageText;
		    				}
		    			});
		    			
		    			if(TTSConfigOptions.readImagesOrdering == 'before') {
		    				text = imagesText + text;
		    			} else {
		    				text = text; + imagesText;
		    			}
		    		}
				} else {
					text = $.trim($(TTSConfigOptions.mainpageSelector).immediateText());  
				}
		    }
		   
		    // Removes carriage return and double spaces
		    text = text.replace(/[\n\r\t]/g, ' ');
		    text = text.replace(/\s+/g, ' ');
		    text = text.substring(0, parseInt(70 + 30));
		    
		    return text;
		}
		
		/**
		 * Visualizza un messaggio informativo all'utente in base allo state passato
		 * che puo rappresentare la fine dei chunk e quindi della lettura oppure 
		 * la mancanza di testo valido da renderizzare come audio
		 * @access private
		 * @param String state
		 * @return Void
		 */
		function showMessages (state) { 
			// Manipolazione del DOM per il div di messages interaction
			switch(state) { 
				case 'playing':
					// Remove precedente inject
					$('div#tts_message').remove();
					$('<div/>').attr('id', 'tts_message').css('background-color', TTSConfigOptions.toolbarBgcolor).prependTo('#accessibility-links').append('<div id="playicon"></div>');
					$('#playicon').css('background-color', TTSConfigOptions.toolbarBgcolor);
					break;
					
				case 'paused':
					// Remove precedente inject
					$('div#tts_message').remove();
					$('<div/>').attr('id', 'tts_message').css('background-color', TTSConfigOptions.toolbarBgcolor).prependTo('#accessibility-links').append('<div id="msgtext">' + fr_paused + '</div>'); 
					break;
					
				case 'finished':
					// Reset all current sound ID
					currentSoundID = null;	 
					$('div#tts_message').fadeOut(500, function(){
						$(this).remove();
					}); 
					break;
					
				case 'notext': 
					$('div#tts_message').remove();
					$('<div/>').attr('id', 'tts_message').css('background-color', TTSConfigOptions.toolbarBgcolor).prependTo('#accessibility-links').append('<div id="msgtext">' + fr_notext + '</div>').delay(1500).fadeOut(500, function(){
						$(this).remove();
					}); 
					break; 
			}
		};
		
		/**
		 * Registrazione event handlers UI 
		 * @method IIFE
		 * @access private
		 * @return Void
		 */
		function registerEvents() {
			// Play actions
			$(document).on('click.frTTS', '#fr_screenreader_play', function(event){ 
				active = true;
				readerStatus = 'play';
				clearTimeout(preloadTimeout);
				// Start cursor
				// Event delegation del click sugli elementi del body con contesto=target
				fullInnerText = findTextToRead(); 
					
				if(!fullInnerText.length) {
					showMessages('notext'); 
				} else {
					// Stop and destroy eventuali riproduzioni
					destroyPlay();
					// Start del play ricorsivo con chunk system
					if(TTSConfigOptions.readerEngine == 'proxy_responsive') {
						responsiveVoice.speak(fullInnerText, 
											  languageCode, 
											  {volume: (TTSConfigOptions.screenReaderVolume / 100),
											   onend: function(){
												   // Remove del div messages 
												   showMessages('finished'); 
											   }
						});
					} else {
						playChunk();
					}
					showMessages('playing');
				}
				$('#fr_screenreader_pause').removeClass('active');
				event.stopPropagation();
				event.preventDefault();  
			}); 
			
			// Stop actions
			$(document).on('click.frTTS', '#fr_screenreader_pause', function(event){ 
				if(!currentSoundID && TTSConfigOptions.readerEngine != 'proxy_responsive') {
					return;
				}
				switch (active) {
					case true:
						active = false; 
						readerStatus = 'pause';
						if(TTSConfigOptions.readerEngine == 'proxy_responsive') {
							responsiveVoice.pause();
						} else {
							soundManager.pause(currentSoundID);
						}
						// Remove del div messages 
						showMessages('paused'); 
						$(this).addClass('active');
					break;
					
					case false:
						active = true; 
						readerStatus = 'play';
						if(TTSConfigOptions.readerEngine == 'proxy_responsive') {
							responsiveVoice.resume();
						} else {
							soundManager.resume(currentSoundID);
						}
						// Remove del div messages 
						showMessages('playing'); 
						$(this).removeClass('active');
					break;
				}
			});
			
			// Stop actions
			$(document).on('click.frTTS', '#fr_screenreader_stop', function(event){ 
				active = false;
				readerStatus = 'stop';
				clearTimeout(preloadTimeout);
				$('#fr_screenreader_pause').removeClass('active');
				// Start cursor
				if(TTSConfigOptions.readerEngine == 'proxy_responsive') {
					responsiveVoice.cancel();
				} else {
					destroyPlay();
				}
				// Remove del div messages 
				showMessages('finished'); 
			});
			
			//Volume actions
			$(document).on('click.frTTS', 'div.volume_step', function(event){ 
				$(this).siblings('div').removeClass('active');
				$(this).prevAll('div').addClass('active');
				$(this).addClass('active');
				soundManager.setVolume(currentSoundID, parseInt($(this).attr('data-value')));
				TTSConfigOptions.screenReaderVolume = $(this).attr('data-value');
			}); 
		}
		
		/**
		 * Init function dummy constructor
		 * @access private
		 * @return Void
		 */
		(function init() {
			// Estensione prototype jQuery object per collezionare solo l'innerText del nodo target
			$.fn.extend({
				immediateText: function() {
					return this.clone().find("*").remove().end().text();
				}
			});
			 
			// Init state
			var bindThis = this;
			active = false; 
			readerStatus = 'stop';
			// Init del TTS engine
			var defaultOptions = {
					baseURI: null,
					toolbarBgcolor: '#EEE',
					token: null,
					langCode: 'en',
					chunkLength: parseInt(70 + 30), 
					screenReaderVolume: 80,
					preload: 0,
					readPage: 1,
					readChildNodes: 1,
					excludeScripts: 1,
					readImages: 0,
					readImagesAttribute: 'alt',
					readImagesOrdering: 'before',
					mainpageSelector: '*[name*=main], *[class*=main], *[id*=main], *[id*=container], *[class*=container]',
					readerEngine: 'proxy_virtual',
					engineToken: 0
				};
			// Init options
			TTSConfigOptions = $.extend({}, defaultOptions, configOptions );
			
			// If the reader engine is the Natural Reader try to request a fresh token
			if(TTSConfigOptions.readerEngine == 'proxy_fallback') {
				$.getJSON("http://api.naturalreaders.com/v2/auth/requesttoken?callback=?",{appid:"de52ydkplza",appsecret:"g13pqbr77k84cggw4oscsggwcos8sco"},null).always(function(response,status,xhr){
					if(status == "success" && response.rst == "true"){
						TTSConfigOptions.engineToken = response.requesttoken;
					}
				});
			}
			
			// Preload image for wave meter
			var tmpImage = new Image();
			tmpImage.src = TTSConfigOptions.baseURI + 'libraries/controller/css/images/waves.gif';
			
			// Storing del langcode
			languageCode = TTSConfigOptions.langCode;
			
			if(TTSConfigOptions.readerEngine == 'proxy_responsive') {
				var responsiveAudioLanguagesMapping = 
									   {'en-US' : 'US English Female',
										'en-AU' : 'Australian Female',
										'en-GB' : 'UK English Female',
										'en' : 'UK English Female',
										'es-ES' : 'Spanish Female',
										'es' : 'Spanish Female',
										'fr-FR' : 'French Female',
										'fr' : 'French Female',
										'de-DE' : 'Deutsch Female',
										'de' : 'Deutsch Female',
										'it-IT' : 'Italian Female',
										'it' : 'Italian Female',
										'el-GR' : 'Greek Male',
										'el' : 'Greek Male',
										'hu-HU' : 'Hungarian Female',
										'hu' : 'Hungarian Female',
										'tr-TR' : 'Turkish Female',
										'tr' : 'Turkish Female',
										'ru-RU' : 'Russian Female',
										'ru' : 'Russian Female',
										'nl-BE' : 'Dutch Female',
										'nl-NL' : 'Dutch Female',
										'nl' : 'Dutch Female',
										'sv-FI' : 'Swedish Female',
										'sv-SE' : 'Swedish Female',
										'sv' : 'Swedish Female',
										'nb-NO' : 'Norwegian Female',
										'nb' : 'Norwegian Female',
										'ja-JP' : 'Japanese Female',
										'ja' : 'Japanese Female',
										'ko-KR' : 'Korean Female',
										'ko' : 'Korean Female',
										'zh-CN' : 'Chinese Female',
										'zh-HK' : 'Chinese Female',
										'zh-MO' : 'Chinese Female',
										'zh-SG' : 'Chinese Female',
										'zh-TW' : 'Chinese Female',
										'zh' : 'Chinese Female',
										'hi-IN' : 'Hindi Female',
										'hi' : 'Hindi Female',
										'sr-BA' : 'Serbian Male',
										'sr-SP' : 'Serbian Male',
										'sr' : 'Serbian Male',
										'hr-BA' : 'Croatian Male',
										'hr-HR' : 'Croatian Male',
										'hr' : 'Croatian Male',
										'bs-BA' : 'Bosnian Male',
										'bs' : 'Bosnian Male',
										'ro-RO' : 'Romanian Male',
										'ro' : 'Romanian Male',
										'ca-ES' : 'Catalan Male',
										'ca' : 'Catalan Male',
										'fi-FI' : 'Finnish Female',
										'fi' : 'Finnish Female',
										'af-ZA' : 'Afrikaans Male',
										'af' : 'Afrikaans Male',
										'sq-AL' : 'Albanian Male',
										'sq' : 'Albanian Male',
										'ar-AR' : 'Arabic Male',
										'ar-AE' : 'Arabic Male',
										'ar-AA' : 'Arabic Male',
										'ar' : 'Arabic Male',
										'hy-AM' : 'Armenian Male',
										'hy' : 'Armenian Male',
										'cs-CZ' : 'Czech Female',
										'cs' : 'Czech Female',
										'da-DK' : 'Danish Female',
										'da' : 'Danish Female',
										'eo' : 'Esperanto Male',
										'is-IS' : 'Icelandic Male',
										'is' : 'Icelandic Male',
										'id-ID' : 'Indonesian Female',
										'id' : 'Indonesian Female',
										'lv-LV' : 'Latvian Male',
										'lv' : 'Latvian Male',
										'mk-MK' : 'Macedonian Male',
										'mk' : 'Macedonian Male',
										'pl-PL' : 'Polish Female',
										'pl' : 'Polish Female',
										'pt-BR' : 'Brazilian Portuguese Female',
										'pt-PT' : 'Portuguese Female',
										'pt' : 'Portuguese Female',
										'sk-SK' : 'Slovak Female',
										'sk' : 'Slovak Female',
										'sw-KE' : 'Swahili Male',
										'sw' : 'Swahili Male',
										'ta-IN' : 'Tamil Male',
										'ta' : 'Tamil Male',
										'th-TH' : 'Thai Female',
										'th' : 'Thai Female',
										'vi-VN' : 'Vietnamese Male',
										'vi' : 'Vietnamese Male',
										'cy-GB' : 'Welsh Male',
										'cy' : 'Welsh Male'
									};
				// Check existance and overwrite
				if(languageCode in responsiveAudioLanguagesMapping) {
					languageCode = responsiveAudioLanguagesMapping[languageCode];
					responsiveVoice.setDefaultVoice(languageCode);
				}
			}
			
			// Storing del limite lunghezza chunk
			limit = parseInt(TTSConfigOptions.chunkLength);
			
			var jchatHasTouch = function() {
				 // Add touch icon
			    var windowOnTouch = !!('ontouchstart' in window);
			    var msTouch = !!(navigator.msMaxTouchPoints);
			    	
			    var hasTouch = windowOnTouch || msTouch;
			    return hasTouch;
			}
			if(jchatHasTouch()) {
				TTSConfigOptions.preload = 0;
			}
			
			// Add volume steps
			addVolumeSteps();
			  
			// Manage background color for the toolbar or buttons only based on the template
			if(TTSConfigOptions.template == 'custom.css') {
				$('#volume_plugin').css('background', TTSConfigOptions.toolbarBgcolor);
			}
			
			// Controller register events
	    	registerEvents();
		}).call(this); 
		
		/**
		 * Restituisce lo stato di attivazione della classe
		 * @access public
		 * @return Boolean
		 */
		this.getState = function() {
			return active;
		};
 
	};
})(jQuery); 