<template>
    <div class="wm-menus-editor">
        <div class="wm-edit-area">
            <div class="wm-preview">
                <div class="wm-header">
                    <span class="wm-text">公众号</span>
                </div>
                <menus :menus.sync="menus" :menu-auto-id.sync="menuAutoId"/>
            </div>
            <div
                v-if="$global.currentMenu"
                class="wm-form"
            >
                <div class="wm-header">
                    <span>{{ $global.currentMenu.name }}</span>
                    <a
                        href="javascript:void(0);"
                        class="wm-pull-right"
                        @click="onRemoveCurrent"
                    >删除菜单</a>
                </div>

                <w-input
                    label="菜单名称"
                    v-model="$global.currentMenu.name"
                    :has-error="hasError('name')"
                    :error-text="getError('name')"
                    inline
                />

                <w-select
                    class="vertical-middle"
                    label="菜单内容"
                    :has-error="hasError('type')"
                    :error-text="getError('type')"
                    v-model="$global.currentMenu.type"
                    inline
                >
                    <option
                        v-for="key of Object.keys(menuTypes)"
                        :key="key"
                        :value="key"
                    >{{ menuTypes[key] }}</option>
                </w-select>

                <div
                    class="wm-content-wrapper"
                    v-show="!currentHasSub"
                >
                    <component
                        :has-error="hasError(this.currentContentField)"
                        :error-text="getError(this.currentContentField)"
                        v-if="currentContentComponent"
                        :is="currentContentComponent"
                        :events="events"
                        :current-v="currentV"
                        :field-errors="fieldErrors.mini"
                    />
                </div>

                <div v-show="currentHasSub" style="margin-top: 20px;">
                    <span class="wm-grey-1">已设置子菜单，只能编辑菜单名</span>
                </div>
            </div>
            <div
                v-else
                class="wm-choose-hint"
            >在左侧选择菜单编辑
            </div>
        </div>
        <div class="wm-footer-toolbar">
            <button
                class="wm-btn wm-btn-primary"
                @click="onSave"
                :disabled="!canSave"
                :title="emptyMenus ? '至少要有一个菜单才能保存' : ''"
            >保存</button>
            <refresh :on-refresh="getData"/>
            <button
                class="wm-btn"
                @click="onReset"
            >重置</button>
        </div>
    </div>
</template>

<script>
import { postResources, getResources } from '@/api/wechat'
import Menus from '@/components/Menus'
import ContentView from '@/components/ContentView'
import ContentEvent from '@/components/ContentEvent'
import ContentMini from '@/components/ContentMini'
import { MENU_TYPES, WECHAT_ERROR_CODES } from '@/common/constants'
import { required, url } from 'vuelidate/lib/validators'
import singleErrorHelper from '@/common/single-error-helper'

/**
 * 手动为无限嵌套的每个菜单单独生成验证
 * @param menus
 */
const buildMenusValidations = menus => {
    const validations = {}
    const validators = {
        name: {
            required,
        },
        type: {
            required,
        },
    }

    menus.forEach((menu, index) => {
        if (menu.sub_button.length > 0) {
            validations[index] = {
                name: validators.name,
                sub_button: buildMenusValidations(menu.sub_button),
            }
        } else {
            const t = { ...validators }

            switch (menu.type) {
                case 'view':
                    t.url = {
                        required,
                        url,
                    }
                    break
                case 'miniprogram':
                    t.appid = {
                        required,
                    }
                    t.pagepath = {
                        required,
                    }
                    t.url = {
                        required,
                    }
                    break
                default:
                    t.key = {
                        required,
                    }
            }

            validations[index] = t
        }
    })

    return validations
}

export default {
    name: 'MenusEditor',
    components: {
        Menus,
        ContentView,
        ContentEvent,
        ContentMini,
    },
    mixins: [
        singleErrorHelper,
    ],
    validations() {
        return {
            menus: buildMenusValidations(this.menus),
        }
    },
    data() {
        return {
            menus: [],
            events: [],
            menuAutoId: 1,
            saving: false,

            fieldErrors: {
                name: {
                    required: '必须填写',
                },
                key: {
                    required: '必须选择',
                },
                type: {
                    required: '必须选择',
                },
                url: {
                    required: '必须填写',
                    url: '必须是有效的 URL',
                },
                mini: {
                    url: {
                        required: '必须填写',
                        url: '必须是有效的 URL',
                    },
                    appid: {
                        required: '必须填写',
                    },
                    pagepath: {
                        required: '必须填写',
                    },
                },
            },
        }
    },
    computed: {
        menuTypes() {
            return MENU_TYPES
        },
        currentHasSub() {
            return this.$global.currentMenu.sub_button.length > 0
        },
        currentContentType() {
            let type = this.$global.currentMenu.type
            switch (type) {
                case 'view':
                    return 'view'
                case 'miniprogram':
                    return 'mini'
                default:
                    return 'event'
            }
        },
        currentContentComponent() {
            let type = this.currentContentType

            return type ? `content-${type}` : null
        },
        currentContentField() {
            switch (this.currentContentType) {
                case 'view':
                    return 'url'
                case 'mini':
                    return 'mini'
                default:
                    return 'key'
            }
        },
        /**
         * 当前激活菜单对应的验证相关数据
         */
        currentV() {
            const indexes = this.$global.currentMenuIndex.split('-')
            let cur = null
            let subs = this.$v.menus
            do {
                let i = indexes.shift()
                cur = subs[i]
                subs = cur.sub_button
            } while (indexes.length)

            return cur
        },
        emptyMenus() {
            return this.menus.length == 0
        },
        canSave() {
            return !this.saving && !this.emptyMenus
        },
    },
    created() {
        this.getData()
    },
    destroyed() {
        // 该组件销毁后，清除全局数据中的，关于当前激活菜单的值
        this.$global.currentMenu = null
        this.$global.currentMenuIndex = null
    },
    methods: {
        getData() {
            this.getMenus()
            this.getEvents()
        },

        async getMenus() {
            const { data } = await getResources('menus')

            if (data.status) {
                this.menus = data.data.menu.button
                this.menusBak = JSON.stringify(this.menus)

                this.menuAutoId = this.addUniqueKey(this.menus)

                this.activeFirstMenu()
            }
        },

        async getEvents() {
            const { data } = await getResources('menu_events')
            this.events = data.data
        },

        /**
         * 给菜单加上唯一标识
         *
         * @param menus
         * @param id
         */
        addUniqueKey(menus, id = 1) {
            menus.forEach(item => {
                item.id = id++
                id = this.addUniqueKey(item.sub_button, id)
            })

            return id
        },
        async onSave() {
            if (!this.canSave) {
                return
            }

            this.$v.$touch()

            if (this.$v.$invalid) {
                this.$notice({
                    msg: '请填写完正确的配置',
                    type: 'error',
                })
                const errorMenu = this.getFirstErrorMenu()
                this.$bus.$emit('menuActive', errorMenu)
                return
            }

            try {
                this.saving = true
                await postResources('menus', this.menus)
            } finally {
                this.saving = false
            }
        },
        onRemoveCurrent() {
            if (!confirm('确认删除？')) {
                return
            }

            const [parent, sub] = this.$global.currentMenuIndex.split('-')

            let nextActive = null

            if (sub === undefined) {
                this.menus.splice(parent, 1)
            } else {
                const parentMenu = this.menus[parent]

                parentMenu.sub_button.splice(sub, 1)
                nextActive = parentMenu
            }

            this.$bus.$emit('menuActive', nextActive)
        },
        activeFirstMenu() {
            // 先清除当前的激活菜单，避免某些依赖于当前菜单数据的计算属性报错
            this.$bus.$emit('menuActive', null)
            this.$nextTick(() => {
                if (this.menus.length == 0) {
                    return
                }

                let menu = null
                const subs = this.menus[0].sub_button
                if (subs.length == 0) {
                    menu = this.menus[0]
                } else {
                    menu = subs[0]
                }

                this.$bus.$emit('menuActive', menu)
            })
        },
        onReset() {
            this.menus = JSON.parse(this.menusBak)
            this.activeFirstMenu()
        },
        getFirstErrorMenu(menus = this.menus, vMenus = this.$v.menus) {
            for (let i = 0; i < menus.length; i++) {
                const menu = menus[i]
                const vMenu = vMenus[i]

                // 如果菜单有错误，则返回他子菜单中的某个错误菜单，或者自己
                if (vMenu.$invalid) {
                    return this.getFirstErrorMenu(menu.sub_button, vMenu.sub_button) || menu
                }
            }

            return null
        },
    },
}
</script>

<style scoped lang="scss">
@import "~@/../sass/vars.scss";

$preview-width: 300px;
$margin-right: 20px;
$form-width: $page-width - $margin-right - $preview-width;

.wm-edit-area {
    height: 500px;
    display: flex;
}

.wm-preview {
    min-width: $preview-width;
    margin-right: $margin-right;
    border: $grey-border;
    display: flex;
    flex-direction: column;
    justify-content: space-between;

    .wm-header {
        height: 50px;
        background: #3a3a3e;
        color: white;
        display: flex;
        justify-content: center;
        align-items: center;
    }
}

.wm-form {
    padding: 0 20px;
    border: $grey-border;
    width: $form-width;
    background-color: #f4f5f9;

    .wm-header {
        height: 40px;
        line-height: 40px;
        border-bottom: $grey-border;
        border-width: 2px;
        margin-bottom: 20px;
    }

    .wm-content-wrapper {
        border: $grey-border;
        background-color: #fff;
        padding: 20px;
    }
}

.wm-choose-hint {
    width: $form-width;
    text-align: center;
    line-height: 600px;
    color: $hint-color;
}

.wm-name-item {
    margin-top: 30px;
}

.wm-footer-toolbar {
    border: none !important;
}
</style>
