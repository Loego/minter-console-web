<script>
    import {mapState} from 'vuex';
    import {validationMixin} from 'vuelidate';
    import required from 'vuelidate/lib/validators/required';
    import minLength from 'vuelidate/lib/validators/minLength';
    import maxLength from 'vuelidate/lib/validators/maxLength';
    import withParams from 'vuelidate/lib/withParams';
    import {createCoin} from "minter-js-sdk/src/coin";
    import checkEmpty from '~/assets/v-check-empty';
    import {getErrorText} from "~/assets/server-error";
    import {getTxUrl} from "~/assets/utils";
    import {NODE_URL} from "~/assets/variables";

    const MIN_CRR = 10;
    const MAX_CRR = 100;

    const crrValidator = withParams({type: 'validCrr'}, function(value) {
        let crr = parseInt(value, 10);
        return MIN_CRR <= crr && MAX_CRR >= crr;
    });

    export default {
        directives: {
            checkEmpty,
        },
        mixins: [validationMixin],
        filters: {
            uppercase: (value) => value.toUpperCase(),
        },
        data() {
            return {
                isFormSending: false,
                serverError: '',
                serverSuccess: '',
                form: {
                    coinName: '',
                    coinSymbol: '',
                    initialAmount: null,
                    crr: null,
                    initialReserve: null,
                    message: '',
                },
            }
        },
        validations: {
            form: {
                coinName: {
                    required,
                    maxLength: maxLength(64),
                },
                coinSymbol: {
                    required,
                    minLength: minLength(3),
                    maxLength: maxLength(10),
                },
                initialAmount: {
                    required,
                },
                crr: {
                    required,
                    between: crrValidator,
                },
                initialReserve: {
                    required,
                },
                message: {
                    maxLength: maxLength(128),
                },

            }
        },
        computed: {
        },
        methods: {
            submit() {
                if (this.isFormSending) {
                    return;
                }
                if (this.$v.$invalid) {
                    this.$v.$touch();
                    return;
                }
                this.isFormSending = true;
                this.serverError = '';
                this.serverSuccess = '';
                this.$store.dispatch('FETCH_ADDRESS_ENCRYPTED')
                    .then(() => {
                        createCoin({
                            nodeUrl: NODE_URL,
                            privateKey: this.$store.getters.privateKey,
                            ...this.form,
                        }).then((response) => {
                            this.isFormSending = false;
                            this.serverSuccess = response.data.result;
                            this.clearForm();
                        }).catch((error) => {
                            console.log(error)
                            this.isFormSending = false;
                            this.serverError = getErrorText(error)
                        })
                    })
                    .catch((error) => {
                        this.isFormSending = false;
                        this.serverError = getErrorText(error)
                    })
            },
            clearForm() {
                this.form.coinName = '';
                this.form.coinSymbol = '';
                this.form.initialAmount = null;
                this.form.crr = null;
                this.form.initialReserve = null;
                this.form.message = '';
                this.$v.$reset();
            },
            getTxUrl,
        }
    }
</script>

<template>
    <form class="panel__section" novalidate @submit.prevent="submit">
        <div class="u-grid u-grid--small u-grid--vertical-margin--small">
            <div class="u-cell u-cell--medium--1-2">
                <label class="form-field" :class="{'is-error': $v.form.coinName.$error}">
                    <input class="form-field__input" type="text" v-check-empty
                           v-model.trim="form.coinName"
                           @blur="$v.form.coinName.$touch()"
                    >
                    <span class="form-field__label">Coin name</span>
                </label>
                <span class="form-field__error" v-if="$v.form.coinName.$dirty && !$v.form.coinName.required">Enter coin name</span>
                <span class="form-field__error" v-if="$v.form.coinName.$dirty && !$v.form.coinName.maxLength">Max 64 letters</span>
            </div>
            <div class="u-cell u-cell--medium--1-2">
                <label class="form-field" :class="{'is-error': $v.form.coinSymbol.$error}">
                    <input class="form-field__input" type="text" v-check-empty
                           v-model.trim="form.coinSymbol"
                           @blur="$v.form.coinSymbol.$touch()"
                    >
                    <span class="form-field__label">Coin symbol</span>
                </label>
                <span class="form-field__error" v-if="$v.form.coinSymbol.$dirty && !$v.form.coinSymbol.required">Enter coin symbol</span>
                <span class="form-field__error" v-if="$v.form.coinSymbol.$dirty && !$v.form.coinSymbol.minLength">Min 3 letters</span>
                <span class="form-field__error" v-if="$v.form.coinSymbol.$dirty && !$v.form.coinSymbol.maxLength">Max 10 letters</span>
            </div>
            <div class="u-cell u-cell--medium--1-3">
                <label class="form-field" :class="{'is-error': $v.form.initialAmount.$error}">
                    <input class="form-field__input" type="text" inputmode="numeric" v-check-empty
                           v-model.number="form.initialAmount"
                           @blur="$v.form.initialAmount.$touch()"
                    >
                    <span class="form-field__label">Initial amount</span>
                </label>
                <span class="form-field__error" v-if="$v.form.initialAmount.$dirty && !$v.form.initialAmount.required">Enter amount</span>
            </div>
            <div class="u-cell u-cell--medium--1-3">
                <label class="form-field" :class="{'is-error': $v.form.crr.$error}">
                    <input class="form-field__input" type="text" inputmode="numeric" v-check-empty
                           v-model.number="form.crr"
                           @blur="$v.form.crr.$touch()"
                    >
                    <span class="form-field__label">Constant reserve ratio</span>
                </label>
                <span class="form-field__error" v-if="$v.form.crr.$dirty && !$v.form.crr.required">Enter CRR</span>
                <span class="form-field__error" v-else-if="$v.form.crr.$dirty && !$v.form.crr.between">CRR should be between 10 and 100</span>
            </div>
            <div class="u-cell u-cell--medium--1-3">
                <label class="form-field" :class="{'is-error': $v.form.initialReserve.$error}">
                    <input class="form-field__input" type="text" inputmode="numeric" v-check-empty
                           v-model.number="form.initialReserve"
                           @blur="$v.form.initialReserve.$touch()"
                    >
                    <span class="form-field__label">Initial reserve</span>
                </label>
                <span class="form-field__error" v-if="$v.form.initialReserve.$dirty && !$v.form.initialReserve.required">Enter reserve</span>
            </div>
            <div class="u-cell">
                <label class="form-field" :class="{'is-error': $v.form.message.$error}">
                    <input class="form-field__input" type="text" v-check-empty
                           v-model.trim="form.message"
                           @blur="$v.form.message.$touch()"
                    >
                    <span class="form-field__label">Message</span>
                </label>
                <span class="form-field__error" v-if="$v.form.message.$dirty && !$v.form.message.maxLength">Max 128 bytes</span>
            </div>
            <div class="u-cell">
                <button class="button button--main button--full" :class="{'is-disabled': $v.$invalid}">Create</button>
                <div class="form-field__error" v-if="serverError">{{ serverError }}</div>
            </div>
            <div class="u-cell" v-if="serverSuccess">
                <strong>Tx sent:</strong> <a class="link--default" :href="getTxUrl(serverSuccess)" target="_blank">{{ serverSuccess }}</a>
            </div>
        </div>
    </form>
</template>