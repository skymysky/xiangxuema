<template>
    <div class="editor">
        <div v-show="$parent.article.editor_type == 'markdown'" class="tip">
            请不要改变img标签的格式（会影响文章发布）
        </div>
        <div v-show="$parent.article.editor_type == 'markdown'" id="editorMd"></div>
        <textarea v-show="$parent.article.editor_type == 'html'" id="editorCk">{{articleContent}}</textarea>
    </div>

</template>
<script>
    //todo:图片放大缩小需要按比例
    const fs = require('fs');
    const path = require('path');
    import img from './mixins/img';
    import md from './mixins/md';
    import ck from './mixins/ck';
    import jna from './mixins/jna';
    const {
        ipcRenderer,
        remote
    } = require('electron');
    export default {
        mixins: [img, md, ck, jna],
        data() {
            return {
                articleContent: null,
                articlePath: null,
                tick: null,
                needSave: false,
            }
        },
        methods: {
            saveContent(cb) {
                if (!this.needSave) {
                    if (cb) cb();
                    return;
                }
                this.bus.$emit('saving');
                if (this.$parent.article.editor_type == "html") {
                    this.articleContent = window.CKEDITOR.instances.editorCk.getData();
                } else {
                    this.articleContent = window.editorMd.getValue();
                }
                fs.writeFileSync(path.join(this.articlePath, "a.data"), this.articleContent, this.$root.rwOption);
                this.needSave = false;
                this.db("articles").update({
                    title: this.$parent.article.title,
                    updated_at: new Date()
                }).where("id", this.$parent.article.id).then(rows => {
                    if (cb) cb();
                });
            },
            focus() {
                if (this.$parent.article.editor_type == "html") {
                    window.CKEDITOR.instances.editorCk.focus();
                } else {
                    window.editorMd.focus();
                }
            },
            destroy() {
                clearInterval(this.tick);
                if (this.$parent.article.editor_type == "html" && CKEDITOR.instances.editorCk) {
                    CKEDITOR.instances.editorCk.destroy();
                }
            },
            hookImgUpload() {
                this.$root.imgUploadCb = (obj) => {
                    if (this.$parent.article.editor_type == "html") {
                        this.imageUploadCk(obj);
                    } else {
                        this.imageUploadMd(obj);
                    }
                    this.needSave = true;
                };
            },
            hookWinQuit() {
                var self = this;
                remote.getCurrentWindow().on('close', (event) => {
                    event.preventDefault();
                    self.saveContent();
                    remote.app.quit();
                })
            },
            getContent() {
                this.articlePath = path.join(remote.app.getPath('userData'), "/xxm/" + this.$parent.article.id);
                this.articleContent = fs.readFileSync(path.join(this.articlePath, "a.data"), this.$root.rwOption);
                if (this.$parent.article.editor_type == "html") {
                    this.$nextTick(()=>{
                        this.initEditorCk();
                    });
                } else {
                    this.initEditorMd();
                }
                this.tick = setInterval(() => {
                    this.saveContent()
                }, this.$root.tickStep);
                this.hookImgUpload();
                this.removeUselessImg();
            }
        },
        mounted() {
            this.hookWinQuit();
        }
    }
</script>
<style scoped lang="scss">
    .editor {
        overflow: hidden;
        flex: 1;
        background: #fff;
        border-top: 1px solid #e5e5e5;
        border-bottom: 1px solid #e5e5e5;
    }

    .mdSelected {
        background: #fff !important;
    }

    @keyframes flash {
        0% {
            color: red;
        }

        20% {
            color: #ccc;
        }

        40% {
            color: red;
        }

        60% {
            color: #ccc;
        }

        80% {
            color: red;
        }

        100% {
            color: #ccc;
        }
    }

    .tip {
        animation: flash 5.6s linear;
        position: absolute;
        z-index: 6;
        line-height: 32px;
        right: 12px;
        color: #ccc;
    }
</style>