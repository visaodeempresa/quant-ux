<script>
import css from 'dojo/css'
import on from 'dojo/on'
import touch from 'dojo/touch'
import lang from 'dojo/_base/lang'
import domGeom from 'dojo/domGeom'
import win from 'dojo/win'
import topic from 'dojo/topic'
import _Color from 'common/_Color'
// import CheckBox from 'common/CheckBox'
import Ruler from 'canvas/Ruler'
import GridAndRuler from 'canvas/GridAndRuler'
import SimpleGrid from 'canvas/SimpleGrid'
import RenderFast from 'canvas/RenderFast'

export default {
    name: 'Render',
    mixins:[_Color, RenderFast],
    data: function () {
		/**
			 * The canvas has the following states:
			 *
			 * 0 = Default
			 *
			 * 1 = Screen DragNDrop
			 *
			 * 2 = Widget DragNDrop
			 *
			 * 3 = Adding new box
			 *
			 * 4 = Resizing entity
			 *
			 * 5 = Container DragNDrop
			 *
			 * 6 = Add Line
			 *
			 * 7 = Add Line 2
			 *
			 * 8 = Copy Style
			 *
			 * 9 = Selection
			 *
			 * 10 = Align
			 *
			 * 11 = StandAlone
		 	 */
      return {
					state: 0,
					isSinglePage: false,
					defaultFontSize: 12,
					canvasFlowWidth: 20000,
					canvasFlowHeight: 10000,
					canvasStartX: -1000,
					canvasStartY: -1000,
					canvasMargin: 0.6,
					moveMode: "ps",
					renderDND: true,
					renderLines: true,
					showDistance: true,
					wireInheritedWidgets: false,
					showAnimation: false,
					showRuler: true,
					hasSelectOnScreen: false,
					gridBackground: {}
        }
    },
    components: {},
    methods: {
		initSize () {
			let height = win.getBox().h
			this.domNode.style.height = `${height}px`
		},

    initRender (){
			this.logger.log(2,"initRender", "enter");
			this.domPos = domGeom.position(this.domNode);

			this.widgetDivs = {};
			this.widgetBackgroundDivs = {};
			this.screenDivs = {};
			this.screenLabels = {};
			this.screenBackgroundDivs = {};
			this.lineSVGs = {};
			this.gridBackground = {};
			this.renderedModels = {};


			this.own(topic.subscribe("matc/canvas/fadeout", lang.hitch(this, "onFadeOut")));
			this.own(topic.subscribe("matc/canvas/fadein", lang.hitch(this, "onFadeIn")));
			this.own(on(this.domNode, "contextmenu", lang.hitch(this, "onContextMenu")));
			this.logger.log(2,"initRender", "exit");
		},

		onFadeOut (){
			css.add(this.container, "MatcCanvasFadeOut");
		},

		onFadeIn (){
			css.remove(this.container, "MatcCanvasFadeOut");
		},

		/**********************************************************************
		 * Messages
		 **********************************************************************/

		showSuccess (msg){
			if(this.message){
				css.add(this.message, "MatcMessageSuccess");
				css.remove(this.message, "MatcMessageError MatcMessageHint");
				this.message.textContent = msg;
				setTimeout(lang.hitch(this,"hideMessage"),3000);
			}
		},

		showError (msg){
			if(this.message){
				css.add(this.message, "MatcMessageError");
				css.remove(this.message, "MatcMessageSuccess MatcMessageHint");
				this.message.textContent = msg;
				setTimeout(lang.hitch(this,"hideMessage"), 3000);
			}
		},

		showHint (msg){
			if(this.message){
				css.add(this.message, "MatcMessageHint");
				css.remove(this.message, "MatcMessageSuccess MatcMessageSuccess");
				this.message.textContent = msg;
				setTimeout(lang.hitch(this,"hideMessage"), 3000);
			}
		},

		hideMessage (){
			if(this.message){
				css.remove(this.message, "MatcMessageSuccess MatcMessageError MatcMessageHint");
			}
		},


		setState (state){
			if(state != this.state){

				this.logger.log(2,"setState", "enter > " + state);

				this.state = state;

				/**
				 * cleanup
				 */
				if(this.domNode){
					css.remove(this.domNode, "MatcCanvasStateAdd MatcCanvasStateDnD MatcCanvasStateBeginLine MatcCanvasStateEndLine MatcCanvasStateCopyStyle");
					//this.onDragCleanup();

					if(state == 1 || state == 2){
						css.add(this.domNode, "MatcCanvasStateDnD");
					}

					if(state == 3){
						css.add(this.domNode, "MatcCanvasStateAdd");
					}

					if(state == 6){
						css.add(this.domNode, "MatcCanvasStateBeginLine");
					}

					if(state == 7){
						css.add(this.domNode, "MatcCanvasStateEndLine");
					}

					if(state == 8){
						css.add(this.domNode, "MatcCanvasStateCopyStyle");
					}
				}

			}
		},


		/**********************************************************************
		 * Container Size
		 **********************************************************************/

		/**
		 * Ugly method to set the height from outside. for some shity
		 * reason this is needed as all div will have just size ==0.
		 */
		setHeight (h){
			this.domNode.style.height = h + "px";
		},

		setViewLines (renderLines){
			this.renderLines = renderLines;
			this.settings.renderLines = renderLines;
			this._setStatus("matcSettings", this.settings );
			this.rerender();
		},

		setShowDistance (value){
			this.showDistance = value;
			this.settings.showDistance = value;
			this._setStatus("matcSettings",this.settings);
		},

		setShowAnimation (value){
			this.showAnimation = value;
			this.settings.showAnimation = value;
			this._setStatus("matcSettings",this.settings);
		},

		moveToScreen (screenID){
			if(this.model && this.model.screens[screenID]){
				var screen = this.model.screens[screenID];
				var winBox = win.getBox();
				var xOffSetScreen = (screen.x+ screen.w/2);
				var xOffSetWindow = (winBox.w/2)+ Math.abs(this.canvasPos.x);
				this.canvasPos.x = this.canvasPos.x + (xOffSetWindow - xOffSetScreen) -100;
				var yOffSetScreen = (screen.y);
				var yOffSetWindow = Math.min(winBox.h/2,200)+ Math.abs(this.canvasPos.y);
				this.canvasPos.y = this.canvasPos.y + (yOffSetWindow - yOffSetScreen) -100;
				this.setContainerPos();
			}
		},

		/**********************************************************************
		 * Fonts
		 **********************************************************************/

		setFonts (fonts) {
			this.logger.log(3,"setFonts", "enter > ", fonts);
			if (fonts) {
				this.attachFontsToDom(fonts)
			}
		},

		/**********************************************************************
		 * Container Size
		 **********************************************************************/

		setContainerSize (){
			this.container.style.height = this.getZoomed(this.canvasPos.h, this.zoom) +"px";
			this.container.style.width = this.getZoomed(this.canvasPos.w, this.zoom) +"px";
			this.frame.style.fontSize = this.getZoomed(this.defaultFontSize, this.zoom)  + "px";
			this.containerSize = {
				h: this.getZoomed(this.canvasPos.h, this.zoom),
				w: this.getZoomed(this.canvasPos.w, this.zoom)
			}
		},

		setContainerPos (ignoreScollUpdate){
			this.domUtil.setXY(this.container, Math.round(this.canvasPos.x), Math.round(this.canvasPos.y))
			if (!ignoreScollUpdate){
				this.updateScrollHandlers();
			}
		},

		isInContainer (obj){
			if(
				(obj.x > 0 && (obj.x + obj.w) < this.getZoomed(this.canvasPos.w, this.zoom)) &&
				(obj.y > 0 && (obj.y + obj.h) < this.getZoomed(this.canvasPos.h, this.zoom))
				){
				return true;
			}
			return false;
		},



		/**********************************************************************
		 * Rendering pipeline
		 **********************************************************************/

		rerender (){
			this.render(this.model);
		},

		/**
		 * Method should be called if the positions of widgets have been
		 * changed, but we did not do an complete rerender. This happens
		 * for instance when widgets are moved.
		 */
		onWidgetPositionChange (zoomedModel) {
			this.logger.log(-1,"onWidgetPositionChange", "enter", zoomedModel);
			this.model = zoomedModel;
			this.renderFactory.setZoomedModel(zoomedModel);
			this.renderFactory.updatePositions(zoomedModel)
		},


		render (model){
			this.logger.log(-1,"render", "enter");
			let renderStart = new Date().getTime();
			try {

				if (this.settings.fastRender) {
					this.renderFlowViewFast(model);
				} else {
					this.renderFlowView(model);
				}

				this.afterRender();

				this.renderComments();

				/**
				 * Also update layer list. The renderFlow might call
				 * select which extends the group with additonal children!
				 */
				this.renderLayerList(model);

				/**
				 * Make sure we continue the add mode
				 */
				this.renderAddCommand();
			} catch(e){
				this.logger.error("render", "ups", e);
				this.logger.sendError(e);
			}
			this.logger.log(0,"render", "exit > " + (new Date().getTime() - renderStart) + 'ms');
		},

		renderFlowView (model){
			this.logger.log(2,"renderFlowView", "enter");

			this.beforeRender();
			this.model = model;
			this.cleanUp();
			this.renderCanvas();

			/**
			 * start real rendering
			 */
			for(let id in model.screens){
				this.renderScreen(model.screens[id]);
			}

			var widgets = this.getOrderedWidgets(model.widgets);
			for(let i=0; i< widgets.length; i++){
				let widget = widgets[i];
				this.renderWidget(widget);
			}

			if(this.renderLines){
				for(let id in model.lines){
					let line = model.lines[id];
					if(!line.hidden){
						this.renderLine(model.lines[id]);
					} else {
						this.logger.log(4,"renderFlowView", "Do not render hidden line > " + id);
					}
				}
			}

			this.wireEvents();
			this.renderSelection();
			this.renderDistance();
			if(this.animate){
				setTimeout(lang.hitch(this,"renderAnimation"),1);
			}
			this.logger.log(3,"renderFlowView", "exit");
		},


		renderDistance (){
			if(this.mode == "distance"){
				if(this._selectWidget){
					this.renderScreenDistance();
				}
			}
		},

		/**********************************************************************
		 * Wiring
		 **********************************************************************/
		reWireEvents (){
			this.logger.log(3,"reWireEvents", "enter");
			this.cleanUpAllListeners();
			this.wireEvents();
		},

		wireEvents (){
			this.logger.log(1,"wireEvents", "enter > " + this.mode);

			this.wireCanvas()

			for(let id in this.model.screens){
				this.wireScreen(id)
			}

			for(let id in this.model.widgets){
				this.wireWidget(id);
			}

			// wire comments;
			this.wireComments();
			/**
			 * FIXME: Wire lines too. Then we can call this in setMode();
			 */
			this.logger.log(4,"wireEvents", "exit");
		},

		/**
		 * wire all widgets that are *NOT* inherited
		 */
		wireWidget (id) {
			let widget = this.model.widgets[id];

			if (this.isElementLocked(widget) || this.isElementHidden(widget)) {
				return
			}

			let div = this.widgetDivs[widget.id];
			if (!widget.inherited || this.wireInheritedWidgets){
				if(this.mode == "edit" || this.mode == "addLine"){
					let locked = widget.style.locked;
					if(locked){
						this.tempOwn(on(div, "mousedown", lang.hitch(this, "onWidgetDndClick", widget.id, div, null)));
					} else {
						this.registerDragOnDrop(div, widget.id, "onWidgetDndStart", "onWidgetDndMove", "onWidgetDndEnd", "onWidgetDndClick");
					}
					this.tempOwn(on(div, touch.over, lang.hitch(this, "setHoverWidget", widget)));
				} else if(this.mode == "view" ){
					this.tempOwn(on(div, "mousedown", lang.hitch(this, "onWidgetDndClick", widget.id, div, null)));
				} else if (this.mode == "distance"){
					this.tempOwn(on(div, touch.over, lang.hitch(this, "renderWidgetDistance", widget)));
					this.tempOwn(on(div, touch.out, lang.hitch(this, "renderWidgetDistance", null)));
					/**
					 * TODO: Make addCloneWork. Really use dull
					 */
					this.tempOwn(on(div, touch.press, lang.hitch(this, "addClonedWidget", widget))); // ALT + DND
				}
			} else {
				this.tempOwn(on(div, "click", lang.hitch(this, "onInheritedWidgetSelected", widget)));
			}
		},

		isElementLocked (widget) {
			return widget && widget.props && widget.props.locked
		},

		isElementHidden (widget) {
			return widget && widget.props && widget.props.hidden
		},

		wireScreen (id) {
			let dndDiv = this.screenDivs[id];
			let screen = this.model.screens[id];

			if (this.isElementLocked(screen)) {
				return
			}

			/**
			 * register dnd
			 */
			if (this.mode == "addLine") {
				this.registerDragOnDrop(dndDiv, screen.id, "onScreenDndStart", "onScreenDndMove", "onScreenDndEnd", "onScreenDndClick");
			} else if(this.mode == "edit" && !this.isSinglePage){
				if (this.hasSelectOnScreen) {
					let lbl = this.screenLabels[id]
					if (lbl) {
						this.registerDragOnDrop(dndDiv, screen.id, "onScreenDndStart", "onScreenDndMove", "onScreenDndEnd", "onScreenDndClick", lbl);
					}
				} else {
					this.registerDragOnDrop(dndDiv, screen.id, "onScreenDndStart", "onScreenDndMove", "onScreenDndEnd", "onScreenDndClick");
				}
			} else if(this.mode == "view"){
				this.tempOwn(on(dndDiv, "mousedown", lang.hitch(this, "onScreenDndClick", screen.id, dndDiv, null)));
			}
		},

		wireCanvas () {
			if(this.moveMode == "classic" && (this.mode == "edit" || this.mode == "view" || this.mode === "data") ){
				/**
				 * In the classic mode the
				 */
				this.registerDragOnDrop(this.container, "container", "onCanvasDnDStart", "onCanvasDnDMove", "onCanvasDnDEnd", "onCanvasDnClick");
			} else if (this.mode === "edit" || this.mode === "view" || this.mode === "data"){
				/**
				 * The must be mousedown, because in chrome the touch press is fired after mousedown and fucks up some how our state maschine
				 */

				this.tempOwn(on(this.container, "mousedown", lang.hitch(this, "onSelectionStarted") ));

			} else if(this.mode == "move"){
				this.registerDragOnDrop(this.container, "container", "onCanvasDnDStart", "onCanvasDnDMove", "onCanvasDnDEnd", null);
			} else if(this.mode == "select"){
				this._selectionToolPressListener = on(this.container,"mousedown", lang.hitch(this,"onSelectionStarted"));
			} else if(this.mode == "hotspot"){
				this._hotspotToolPressListener = on(this.container,"mousedown", lang.hitch(this,"onToolHotspotStart"));
			} else if(this.mode == "addLine") {
				this.registerDragOnDrop(this.container, "container", "onCanvasDnDStart", "onCanvasDnDMove", "onCanvasDnDEnd", "onCanvasDnClick");
			} else if(this.mode == "addText") {
				this._hotspotToolPressListener = on(this.container,"mousedown", lang.hitch(this,"onToolTextStart"));
			} else if(this.mode == "addBox") {
				this.onToolBoxInit();
				this._hotspotToolPressListener = on(this.container,"mousedown", lang.hitch(this,"onToolBoxStart"));
			}
		},

		renderCanvas (){
			this.initSVG();
			this.setContainerSize();
			this.setContainerPos();
			this.setZoomedContainerPosition();
			this.renderFactory.setScaleFactor(this.zoom, this.zoom);
			this.renderFactory.setZoomedModel(this.model);
		},

		beforeRender () {
		},

		/**
		 * add divs back to dom!
		 */
		afterRender (){
			if(this._afterRenderCallBack ){
				/**
				 * Call the callback to make sure it is not running in request animationframe
				 * FIXME: I had now once the issue that the callback was null or so beucase the
				 * edit was closed....
				 */
				setTimeout(this._afterRenderCallBack, 50);
			}

			if(this._afterRenderCallBackSync){
				this._afterRenderCallBackSync()
			}
			delete this._afterRenderCallBack ;
			delete this._afterRenderCallBackSync
		},


		addAfterRenderCallBack (fct, isSync = false){
			if (!isSync) {
				this._afterRenderCallBack = fct;
			} else {
				this._afterRenderCallBackSync = fct
			}
		},

		renderScreen (screen){
			this.logger.log(4,"renderScreen", "enter");

			var dndDiv = null;
			var backgroundDiv = null;
			/**
			* We need these div also to check if the screen was rendered!
			*/
			if(!this.screenDivs[screen.id]){
				/**
				 * create dnd
				 */
				dndDiv = this.createScreenDnD(screen);
				this.screenDivs[screen.id] = dndDiv;
				this.widgetContainer.appendChild(dndDiv);

				var lbl = document.createElement("div");
				css.add(lbl, "MatcScreenLabel");
				this.setTextContent(lbl, screen.name);
				this.screenLabels[screen.id] = lbl;
				dndDiv.appendChild(lbl);

				/**
				 * Create a background box
				 */
				backgroundDiv = this.createScreen(screen);
				this.screenContainer.appendChild(backgroundDiv);
				this.screenBackgroundDivs[screen.id] = backgroundDiv;
			}

			/**
			 * set style and grid
			 */
			this.renderFactory.setStyle(backgroundDiv, screen);
			this.renderGrid(dndDiv, screen);
			this.renderScreenButtons(dndDiv, screen)

			return dndDiv;
		},


		renderWidget (widget){
			this.logger.log(4,"renderWidget", "enter");

			/**
				* check if we have to create div again.. Also used as indicator
				* if the widget was rendered!
				*/
			var div = null;
			if(!this.widgetBackgroundDivs[widget.id] && !this.isElementHidden(widget)){

				/**
					* create dnd
					*/
				if (this.renderDND && !this.isElementLocked(widget)) {

					div = this.createWidgetDnD(widget);
					if(widget.inherited){
						css.add(div, "MatcWidgetDNDInherited");
					}
					this.widgetDivs[widget.id] = div;
					this.widgetContainer.appendChild(div);
				}

				/**
					* Create background
					*/
				let divBack = this.createWidget(widget);
				this.screenContainer.appendChild(divBack);
				this.widgetBackgroundDivs[widget.id] = divBack;
			}
			return div;
		},

		renderScreenButtons () {
			/**
			 * Methdod to be implemented by mixins
			 */
		},


		/**************************************************
		 * CleanUp Code
		 **************************************************/

		cleanUp (){
			this.logger.log(3,"cleanUp", "enter");

			if (this.settings && this.settings.fastRender) {
				return this.cleanUpFast();
			}

			/**
			 * Make sure inline edit is flushed
			 */
			this.inlineEditStop();
			this.cleanUpComments();
			this.cleanUpScreenButtons();

			/**
			 * Cleanup any stuff from the zoom
			 */
			this.cleanUpZoom();
			this.cleanUpAllListeners();
			this.cleanUpAlignment();

			this.widgetDivs = {};
			this.widgetBackgroundDivs = {};
			this.screenDivs = {};
			this.screenLabels = {};
			this.screenBackgroundDivs = {};
			this.lineSVGs = {};
			this.gridBackground = {};

			css.remove(this.container, "MatcCanvasFadeOut");
			css.remove(this.container, "MatcCanvasModeAlign");
			css.remove(this.container, "MatcCanvasModeReplicate");

			//this.screenContainer.innerHTML = "";
			//this.widgetContainer.innerHTML = "";
			this.domUtil.removeAllChildNodes(this.screenContainer)
			this.domUtil.removeAllChildNodes(this.widgetContainer)
			this.renderFactory.cleanUp();
			this.cleanUpLines();
			this.cleanUpDebugLines();
			window.scrollTo(0, 0);

			if (this.cleanUpDistributionHandlers) {
				this.cleanUpDistributionHandlers()
			}
		},



		cleanUpAllListeners (){
			this.cleanUpDragNDropListenerListener();
			this.cleanUpTempListener();
			if(this._canvasClickListener){
				this._canvasClickListener.remove();
				delete this._canvasClickListener;
			}
			this.cleanUpSelectionListener();
			if(this._selectionToolPressListener){
				this._selectionToolPressListener.remove();
				delete this._selectionToolPressListener;
			}
			if(this._hotspotToolPressListener){
				this._hotspotToolPressListener.remove();
				delete this._hotspotToolPressListener;
			}
			this.cleanUpResizeHandles();
			this.cleanUpAddNDrop();
		},

		cleanUpScreenButtons () {
			// console.warn('cleanUpScreenButtons() > Not Implemented')
			/**
				* Methdod to be implemented by mixins
				*/
		},

			/**************************************************
		 * New wiring methods?
		 **************************************************/

		addCanvasEventHandler (id, handler) {
			if (!id) {
				console.error('addCanvasEventHandler() > no id passed', id, new Error().stack)
			}
			if (!this._canvasEventHandlers) {
				this._canvasEventHandlers = {}
			}
			if (!this._canvasEventHandlers[id]) {
				this._canvasEventHandlers[id] = []
			}
			this._canvasEventHandlers[id].push(handler)
		},

		removeCanvasEventHandler (id) {
			if (this._canvasEventHandlers && this._canvasEventHandlers[id]) {
				let handlers = this._canvasEventHandlers[id];
				handlers.forEach(handler => {
					handler.remove()
				})
				this._canvasEventHandlers[id] = []
			}
		},

		cleanCanvasEventHandler () {
			for (let id in this._canvasEventHandlers) {
				this.removeCanvasEventHandler(id)
			}
		},

		/**********************************************************************
		 * Create methods that assemble box stuff
		 **********************************************************************/


		renderGrid (backgroundDiv){
			if(this.model.grid && this.model.grid.visible){

				if (this.model.grid.type === "columns"){
					let z = this.zoom + "";
					let h = this.getZoomed(this.zoom,this.model.grid.h);
					let w = this.getZoomed(this.zoom,this.model.grid.w);

					if (!this.gridBackground[z]){
						let columnCount = this.model.grid.columnCount;
						let columnOffset = (this.zoom * this.model.grid.columnOffset);
						let columnGutter = (this.zoom * this.model.grid.columnGutter);
						let columnWidth = (this.zoom * this.model.grid.columnWidth);

						let c = document.createElement("canvas");
						c.width = this.model.screenSize.w;
						c.height = 1;
						let context = c.getContext("2d");

						var lastX = columnOffset;
						for (let i=0; i< columnCount; i++){
							let x = lastX + columnWidth;
							// if gutter is 0, we just draw some lines...
							if (columnGutter > 0 ) {
								context.moveTo(Math.round(lastX), 0);
								context.lineTo(Math.round(x), 0);
							} else {
								context.moveTo(Math.round(lastX), 0);
								context.lineTo(Math.round(lastX +1), 0);
								if (i === (columnCount-1)){
									context.moveTo(Math.round(x-1), 0);
									context.lineTo(Math.round(x), 0);
								}
							}
							lastX = x + columnGutter;
						}
						/**
							* We had here somehow a ghost h...
							*/
						context.moveTo(0, h);
						context.lineTo(w, h);
						context.strokeStyle = this.model.grid.color;
						context.stroke();
						let url = "url(" + c.toDataURL("image/png")  + ")";
						this.gridBackground[z] = url;
					}
					backgroundDiv.style.backgroundImage = this.gridBackground[z]
				} else {
					/**
						* FIXME: We should render the background image for the x times so it is even.
						* e.g. h= 2.5 so we would have to render for 5. Also we should store the image and do
						* the rendering only once...
						*
						* Check drigRuler how we calculate the grid in there
						* ..
						*/
					let h = this.getZoomed(this.zoom,this.model.grid.h);
					let w = this.getZoomed(this.zoom,this.model.grid.w);

					if(w > 0 && h > 0 && w < this.model.screenSize.w && h < this.model.screenSize.h ){
						let z = this.zoom + "";
						if (!this.gridBackground[z]){
							let c= document.createElement("canvas");
							c.width=w;
							c.height=h;
							let context = c.getContext("2d");
							if(this.model.grid.style=="line"){
								context.moveTo(w, 0);
								context.lineTo(w, h);
								context.moveTo(0, h);
								context.lineTo(w, h);
								context.strokeStyle = this.model.grid.color;
								context.stroke();
							} else {
								context.moveTo(w-1, h);
								context.lineTo(w, h);
								context.strokeStyle = this.model.grid.color;
								context.stroke();
							}
							this.gridBackground[z] = "url(" + c.toDataURL("image/png")  + ")";
						}
						backgroundDiv.style.backgroundImage = this.gridBackground[z]
					}
				}
			} else {
				backgroundDiv.style.backgroundImage = 'none'
			}
        },

		createScreenDnD (screen){
			this.logger.log(4,"createScreenDnD", "enter");
			var div = this.createBox(screen);
			css.add(div, "MatcScreenDnD");
			return div;
		},

		createScreen (screen){
			this.logger.log(4,"createScreen", "enter");
			var div = this.createBox(screen);
			css.add(div, "MatcScreen");
			return div;
		},


		createWidgetDnD (widget){
			this.logger.log(4,"createWidgetDnD", "enter");
			var div = this.createBox(widget);
			css.add(div, "MatcWidgetDND");
			if (this.hasLogic(widget)){
				css.add(div, "MatcLogicWidgetDnD");
			}
			/**
			 * Since 2.1.6 we have the data view and need a callback
			 */
			this.createWidgetDataView(widget, div)
			return div;
		},

		createWidgetDataView () {
			// child classes can implement
		},

		createWidget (widget){
			this.logger.log(4,"createWidget", "enter");
			var div = this.createBox(widget);
			css.add(div, "MatcWidget");

			this.renderFactory.createWidgetHTML(div, widget);

			if(this.hasLine(widget)){
				css.add(div, "MatcWidgetWithTransition");
			}

			return div;
		},

		createBox (box){
			this.logger.log(6,"createBox", "enter");
			var div = document.createElement("div");
			this.domUtil.setBox(div, box)
			css.add(div, "MatcBox");
			return div;
		},

		updateBox (box, div){
			this.domUtil.setBox(div, box)
			return div;
		},


		setWidgetPosition (id, pos){
			var widget = this.model.widgets[id];
			if(widget){
				widget.x = pos.x;
				widget.y = pos.y;
				widget.w = pos.w;
				widget.h = pos.h;
				var div = this.widgetBackgroundDivs[id];
				if(div){
					this.updateBox(widget, div);
				}
				div = this.widgetDivs[id];
				if(div){
					this.updateBox(widget, div);
				}
			}
		},

		setScreenPosition (){
		},

		setTempWidgetStyle (id, style){
			//this.logger.log(4,"setTempWidgetStyle", "enter");
			var widget = this.model.widgets[id];
			var div = this.widgetBackgroundDivs[id];
			if(widget && widget.style && div){
				/**
				 * We merge the new style into the current style
				 */
				for (var k in style) {
					widget.style[k] = style[k];
				}
				this.renderFactory.setStyle(div, widget, true);
				this.setCopyStyle(widget, true);
			} else {
				console.warn("setTempWidgetStyle() > Cannot set widget style", id, style);
			}
		},

		setWidgetStyle (id, style, model){
			this.logger.log(-1,"setWidgetStyle", "enter > ", id);
			var widget = this.model.widgets[id];
			var div = this.widgetBackgroundDivs[id];
			if (widget && div){
				/**
				 * Flush inlineEdit if needed
				 */
				var newLabel = this.inlineEditStop();
				if (newLabel && model.props) {
					/**
						* For some reason this will overwrite the style change in the undo()
						* This we live if this...
						*/
					//console.debug("overwrite inline", newLabel)
					//model.props.label = newLabel;
				}
				this.renderFactory.setStyle(div, model, true);
				this.setCopyStyle(widget, false);
			} else {
				console.warn("setWidgetStyle() > Cannot set widget style", id, style);
			}
		},

		/**
		 * copy style to copies (from master screen)
		 */
		setCopyStyle (widget, isTempUpdate) {
			if (widget.copies){
				for(let i=0; i< widget.copies.length; i++){
					let copyID = widget.copies[i];
					let copyWidget = this.model.widgets[copyID];
					let copyDiv = this.widgetBackgroundDivs[copyID];
					if(copyWidget && copyDiv){
						copyWidget.style = widget.style;
						copyWidget.props = widget.props;
						this.renderFactory.setStyle(copyDiv, copyWidget);
					}
				}
			}

			/**
			 * Since 2.2.0 we have screen segments
			 */
			if (widget.segmentParent) {
				this.renderFactory.updateSegementChild(widget, this.model);
			}

			if (widget.container) {
				this.renderFactory.updateContainerChild(widget, this.model);
			}


			if(widget.inheritedCopies){
				for(let i=0; i< widget.inheritedCopies.length; i++){
					/**
						* Here we get also the latest updated model method
						*/
					let copyID = widget.inheritedCopies[i];
					let copyWidget = this.model.widgets[copyID];
					if (isTempUpdate) {
						/**
							* Attention: If this code is called
							* from the setTempWidgetStyle() method, the copyWidget
							* has off course already a style, because it was set
							* in the Layout.createInheritedModel() method.
							*
							* Thus the Layout.mixin() method will not detect the
							* that for instance the background is inherited...
							*
							* Thereefore we have a special update methid here
							*/
						copyWidget.style = this.mixinNotOverwriten(widget.style, copyWidget.style)
						copyWidget.props = this.mixinNotOverwriten(widget.props, copyWidget.props)
					} else {
						/**
							* This code is called when the setTempWidgetStyle() is done. If we would
							* call the normal mixin method it would set an empty _mixin. We do not that,
							* so we ignore that (and copy the old _mixin)
							*/
						copyWidget.style = this.mixin(widget.style, copyWidget.style, false)
						copyWidget.props = this.mixin(widget.props, copyWidget.props, false)
					}

					let copyDiv = this.widgetBackgroundDivs[copyID];
					if(copyWidget && copyDiv){
						this.renderFactory.setStyle(copyDiv, copyWidget);
					}
				}


			}
		},

		setScreenStyle (id){
			var screen = this.model.screens[id];
			var div = this.screenBackgroundDivs[id];
			if(screen && div){
				this.renderFactory.setStyle(div, screen);
				/**
					* Update label as well
					*/
				if(this.screenLabels[id]){
					this.setTextContent(this.screenLabels[id], screen.name);
				}
			} else {
				this.logger.error("setScreenStyle","No screen div for " + id);
				this.logger.sendError(new Error("No Screen Div in setScreenStyle"))
			}
		},

		setTempScreenStyle (id, style){
			var screen = this.model.screens[id];
			var div = this.screenBackgroundDivs[id];
			if(screen && div){
				for (var k in style) {
					screen.style[k] = style[k];
				}
				this.renderFactory.setStyle(div, screen);
			} else {
				this.logger.error("setScreenStyle","No screen div for " + id);
				this.logger.sendError(new Error("No Screen Div in setScreenStyle"))
			}
		},


		/***************************************************************************
			* Mouse Functons
			***************************************************************************/

		getCanvasMousePosition (e){
			var pos = this._getMousePosition(e);
			pos.x -= (this.domPos.x + this.canvasPos.x);
			pos.y-= (this.domPos.y + this.canvasPos.y);
			return pos;
		},

		getRelCanvasMousePosition (e){
			var pos = this.getCanvasMousePosition(e);
			pos.x = pos.x / this.getZoomed(this.canvasPos.w, this.zoom);
			pos.y = pos.y / this.getZoomed(this.canvasPos.h, this.zoom);
			return pos;
		},


		/***************************************************************************
			* Align
		***************************************************************************/

		alignmentShowDistribution (distances){

			if (this._alignmentTool && this._alignmentTool.showDistribution){
				this._alignmentTool.showDistribution(distances)
			}
		},

		alignmentStart (selectedType, selectedModel, activePoint, ignoreIds, showDimensions){
			this.logger.log(1,"alignmentStart","enter > " + selectedType);

			/**
			 * Use the grid only when widget is selected and grid is specified
			 */
			if (this.model.grid) {
				if ("widget" == selectedType || "boundingbox" == selectedType || "group" == selectedType ||  "multi" == selectedType) {
					this._alignmentTool = new GridAndRuler();
					this._alignmentTool.ignoreGroup = this._dragNDropIgnoreGroup;
					this._alignmentTool.showDndDistance = this.showDistance;
					this._alignmentTool.showDimensions = showDimensions;
					if(ignoreIds){
						this._alignmentTool.ignoreIds = ignoreIds;
						this._alignmentTool.setSelectedIDs(ignoreIds);
					}
					this._alignmentTool.start(this, selectedType, selectedModel, activePoint, this.model.grid, this.zoom);
				} else if("grid" == selectedType ) {

					this._alignmentTool = new SimpleGrid();
					this._alignmentTool.start(this, this.model.grid, this.zoom, "RightDown");
					this._alignmentTool.showDimensions = showDimensions;
				} else {
					this._alignmentTool = new Ruler();
					this._alignmentTool.start(this,selectedType, selectedModel,activePoint);
				}
			} else  {

				this._alignmentTool = new Ruler();
				this._alignmentTool.start(this,selectedType, selectedModel,activePoint);
			}
			this._alignmentToolInited = true;
		},


		allignPosition (pos, e){
			if(this._alignmentTool){
				var mouse = this.getCanvasMousePosition(e);
				return this._alignmentTool.correct(pos, e, mouse);
			}
			return pos;
		},

		cleanUpAlignment (){
			this.logger.log(4,"cleanUpAlignment","enter");

			if(this._alignmentTool){
				this._alignmentTool.cleanUp();
				delete this._alignmentTool;
			}

			this._alignmentToolInited = false;

		},

		getModelPosition (){

		},


		/***************************************************************************
		 * Cancel stuff
		 ***************************************************************************/
		/**
			* Register a call back that will be called in case
			* an cancel action is executed. The name of the
			* method has to be passed. The methods can return true,
			* to request an rerendering.
			*/
		setCanvasCancelCallback (l){
			this._cancelCallback = l;
		},

		cleanUpCancelCallbacks (){
			this._cancelCallback = null;
		},

		onCancelAction (){

			this.logger.log(0,"onCancelAction", "enter > " + this._cancelCallback);

			if(this._cancelCallback && this[this._cancelCallback]){

				var rerender = this[this._cancelCallback]();
				if(rerender){
					this.rerender();
				}
			} else {
				this.rerender();
			}

			this._cancelCallback = null;
		},



		/***************************************************************************
			* Helper
			***************************************************************************/
		getColor: function(value){

			if(value == 0){
				return this.defaultLineColor;
			}

			return this.mixColor(value);
		},


		getLastMousePos () {
			return this._lastMousePos;
		},

		setHoverWidget (w){
			this._lastHoverWidget = w;
		}
    },
    mounted () {
    }
}
</script>