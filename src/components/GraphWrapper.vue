<template>
    <div>
        <div id="container"></div>
        <div id="sidebar">
            <EditForm ref="form" @change="changeObjectValues" :cell-data="currentCell"/>
        </div>
    </div>
</template>

<script>
    import mxgraph from 'mxgraph';
    import EditForm from "./EditForm";

    // если планируется использовать встроенные интерфейсы то нужно указать пути к ресурсам
    const graphConfig = {
        mxBasePath: '/mx/', //Specifies the path in mxClient.basePath.
        mxImageBasePath: '/mx/images', // Specifies the path in mxClient.imageBasePath.
        mxLanguage: 'en', // Specifies the language for resources in mxClient.language.
        mxDefaultLanguage: 'en', // Specifies the default language in mxClient.defaultLanguage.
        mxLoadResources: false, // Specifies if any resources should be loaded.  Default is true.
        mxLoadStylesheets: false, // Specifies if any stylesheets should be loaded.  Default is true
    };

    const {
        mxClient, mxUtils, mxEvent, mxEditor, mxRectangle, mxGraph, mxGeometry, mxCell,
        mxImage, mxDivResizer, mxObjectCodec, mxCodecRegistry, mxConnectionHandler
    } = mxgraph(graphConfig);

    window.mxClient = mxClient;
    window.mxUtils = mxUtils;
    window.mxRectangle = mxRectangle;
    window.mxGraph = mxGraph;
    window.mxEvent = mxEvent;
    window.mxGeometry = mxGeometry;
    window.mxCell = mxCell;
    window.mxImage = mxImage;
    window.mxEditor = mxEditor;
    window.mxDivResizer = mxDivResizer;
    window.mxObjectCodec = mxObjectCodec;
    window.mxCodecRegistry = mxCodecRegistry;
    window.mxConnectionHandler = mxConnectionHandler;

    var editor;

    // CustomUserObject
    window.CustomUserObject = function (name, type) {
        this.name = name || 'New Name';
        this.type = type || 'New Type';
        this.clone = function () {
            return mxUtils.clone(this);
        };
    };

    export default {
        name: 'graph-wrapper',
        components: {EditForm},
        data() {
            return {
                currentCell: null
            }
        },
        methods: {
            // изменение выбранных в отдельной панели
            // перенос объекта в среду VUE
            selectionChanged() {
                let cell = editor.graph.getSelectionCell();
                this.$set(this, 'currentCell', cell);
            },
            // возврат обновленного значения в среду mxGraph
            changeObjectValues(newCellValue) {
                editor.graph.model.setValue(this.currentCell, newCellValue);
            },
            // добавление элементов sidebar
            addSidebarIcon: function (graph, sidebar, prototype) {
                let funct = function (graph, evt) {
                    graph.stopEditing(false);

                    let pt = graph.getPointForEvent(evt);

                    let parent = graph.getDefaultParent();
                    let model = graph.getModel();

                    let v1 = model.cloneCell(prototype);

                    model.beginUpdate();
                    try {
                        v1.geometry.x = pt.x;
                        v1.geometry.y = pt.y;
                        v1.style = editor.graph.stylesheet.getDefaultEdgeStyle();
                        v1.geometry.alternateBounds = new mxRectangle(0, 0, v1.geometry.width, v1.geometry.height, '');

                        graph.addCell(v1, parent);
                    } finally {
                        model.endUpdate();
                    }

                    graph.setSelectionCell(v1);
                };

                // Creates the image which is used as the sidebar icon (drag source)
                let wrapper = document.createElement('div');
                wrapper.style.cursor = 'pointer';
                wrapper.style.backgroundColor = '#c3d9ff';
                wrapper.style.margin = '10px';
                wrapper.style.width = '200px';
                wrapper.style.height = '50px';
                wrapper.style.textAlign = 'center';
                wrapper.style.display = 'flex';
                wrapper.style.flexWrap = 'wrap';
                wrapper.style.alignItems = 'center';
                wrapper.style.justifyContent = 'center';
                wrapper.style.border = '2px dashed crimson';
                wrapper.innerHTML = '<div>Custom User Object</div><div style="color: #8C8C8C">Drag me to scheme!</div>';
                sidebar.appendChild(wrapper);

                // Creates the image which is used as the drag icon (preview)
                let dragImage = wrapper.cloneNode(true);
                mxUtils.makeDraggable(wrapper, graph, funct, dragImage);
            },
            // создание и настройка редактора
            createGraph() {
                // Checks if the browser is supported
                if (!mxClient.isBrowserSupported()) {
                    // Displays an error message if the browser is not supported.
                    mxUtils.error('Browser is not supported!', 200, false);
                } else {
                    mxConnectionHandler.prototype.connectImage = new mxImage(require('../assets/handle-connect.png'), 16, 16);

                    let container = document.getElementById('container');
                    container.style.position = 'absolute';
                    container.style.overflow = 'hidden';
                    container.style.left = '250px';
                    container.style.top = '0px';
                    container.style.right = '0px';
                    container.style.bottom = '0px';
                    container.style.background = `url("${require('../assets/grid.gif')}")`;

                    let sidebar = document.getElementById('sidebar');
                    sidebar.style.position = 'absolute';
                    sidebar.style.overflow = 'hidden';
                    sidebar.style.padding = '0px';
                    sidebar.style.left = '0px';
                    sidebar.style.top = '0px';
                    sidebar.style.width = '250px';
                    sidebar.style.bottom = '0px';
                    sidebar.style.display = 'flex';
                    sidebar.style.flexDirection = 'column-reverse';
                    sidebar.style.alignItems = 'center';
                    sidebar.style.justifyContent = 'flex-end';
                    sidebar.style.backgroundColor = '#292961';

                    if (mxClient.IS_QUIRKS) {
                        document.body.style.overflow = 'hidden';
                        new mxDivResizer(container);
                        new mxDivResizer(sidebar);
                    }

                    editor = new mxEditor();
                    editor.setGraphContainer(container);

                    // глобальное управление подключениями
                    editor.graph.setConnectable(true);
                    editor.graph.setCellsDisconnectable(true);
                    editor.graph.setPanning(true);
                    editor.graph.setAllowDanglingEdges(false);

                    // изменение выбранных в отдельной панели
                    editor.graph.getSelectionModel().addListener(mxEvent.CHANGE, () => {
                        this.selectionChanged();
                    });
                    this.selectionChanged();

                    editor.graph.centerZoom = false;
                    editor.graph.swimlaneNesting = false;
                    editor.graph.dropEnabled = true;

                    // Fields are dynamically created HTML labels
                    editor.graph.isHtmlLabel = function (cell) {
                        return !this.isSwimlane(cell) &&
                            !this.model.isEdge(cell);
                    };

                    // not editable
                    editor.graph.isCellEditable = function () {
                        return false
                    };

                    // Returns the name propertie of the user object for the label
                    editor.graph.convertValueToString = function (cell) {
                        if (cell.value != null && cell.value.name != null) {
                            return cell.value.name;
                        }
                        return mxGraph.prototype.convertValueToString.apply(this, arguments); // "supercall"
                    };

                    // Creates a dynamic HTML label for properties
                    editor.graph.getLabel = function (cell) {
                        // console.log('getLabel ', cell);
                        if (cell && this.isHtmlLabel(cell) && cell.value) {
                            // TODO тут код html для содержимого Field
                            let label = '';
                            label += '<div style="width: 100%;display: flex; justify-content: space-between;align-items: center">';
                            label += '<div style="width: 100px;margin-left: 10px; font-weight: 600">' +
                                '<strong>' + mxUtils.htmlEntities(cell.value.name, false) + '</strong>' +
                                '</div>';
                            label += '<div style="width: 100px;margin-right: 10px"><strong style="opacity: 0.7">' + mxUtils.htmlEntities(` \t ${cell.value.type}`, false) + '</strong></div>';
                            label += '</div>';

                            return label
                        }

                        return mxGraph.prototype.getLabel.apply(this, arguments); // "supercall"
                    };

                    // Adds sidebar icon for the propertie object
                    let customObject = new window.CustomUserObject();

                    let object = new mxCell(customObject, new mxGeometry(0, 0, 200, 50), '');
                    object.setVertex(true);
                    object.setConnectable(true);

                    this.addSidebarIcon(editor.graph, sidebar, object);
                }
            },
            // настройка параметров
            init() {
                //////////////////////////////////////////
                // КОДЕКИ КАСТОМНЫХ ПОЛЕЙ для импорта экспорта
                //////////////////////////////////////////
                let codecCustomUserObject = new mxObjectCodec(new window.CustomUserObject());
                codecCustomUserObject.encode = function (enc, obj) {
                    let node = enc.document.createElement('CustomUserObject');
                    mxUtils.setTextContent(node, JSON.stringify(obj));

                    return node;
                };
                codecCustomUserObject.decode = function (dec, node) {
                    let obj = JSON.parse(mxUtils.getTextContent(node));
                    let beatyObj = new window.CustomUserObject();
                    obj = Object.assign(beatyObj, obj);
                    return obj;
                };
                mxCodecRegistry.register(codecCustomUserObject);

                this.createGraph();
            },
        },
        mounted() {
            this.init();
        }
    }
</script>
