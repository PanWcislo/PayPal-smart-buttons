<template>
<div class="relative bg-white dark:bg-dark-900">
    <div
        id="paypal-button-container"
        class="w-full"
        :class="[
            isValid ? '' : 'opacity-40'
        ]"
    ></div>
</div>
</template>

<script>
import { computed, ref, watch } from '@vue/runtime-core';
import { usePage } from '@inertiajs/inertia-vue3';
import __ from "@/helper/trans";
import Spinner from 'UI/Spinner';

import { loadScript } from "@paypal/paypal-js";
import Button from 'UI/Button';
import axios from "axios";
import { isServer, _route, hasRole } from '@/helper/function';
import { Inertia } from '@inertiajs/inertia';

import Payments from '@/Components/Checkout/Payments';

const isProductionMode = process.env.NODE_ENV === "production";
const PAYPAL_APP_CLIENT_ID = process.env.MIX_PAYPAL_APP_CLIENT_ID
const PAYPAL_APP_CLIENT_ID_LIVE = process.env.MIX_PAYPAL_APP_CLIENT_ID_LIVE

export default {
    components: {
        Spinner,
        Button,

        Payments
    },
    emits: ['click-payment-button'],
    props: {
        total: [Number, String],
        discount: [Number, String],
        form: [Object, Array],
        items: [Array],
        subscription: {
            type: [Boolean, String],
            default: false
        }
    },
    setup(props) {
        const PAY_PAL_ACTION = ref(null);
        const loading = ref(false);
        const isValid = ref(false);
        const selectedPayment = ref('paypal');
        const orderID = ref(null);
        const products = computed(() => {
            const items = [];

            props.items.forEach((el) => {
                const product_total_price = Number((el.product ? el.product.item_price : el.item_price).toFixed(2));

                const product = {
                    name: el.product ? el.product.title : (el.name ? el.name : el.title),
                    unit_amount: {
                        currency_code: props.form.currency,
                        value: Number((product_total_price).toFixed(2))
                    },
                    quantity: (el.quantity ? el.quantity : 1),
                    sku: `${(el.product ? el.product.type : el.type)}-${(el.product_id ? el.product_id : el.id)}`,
                    category: "DIGITAL_GOODS"
                }

                items.push(product);
            })

            return items;
        })

        watch(props.form, (val, old) => {
            let checkValid = true;
            for (const [key] of Object.entries(val.data())) {
                if(!isServer && document.getElementById(key)){
                    const element = document.getElementById(key);
                    if(element.required && !element.checkValidity()){
                        checkValid = false;
                        break;
                    }
                }
            }

            if(checkValid){
                isValid.value = true;
                if(PAY_PAL_ACTION.value) PAY_PAL_ACTION.value.enable();
            }
            else {
                isValid.value = false;
                if(PAY_PAL_ACTION.value) PAY_PAL_ACTION.value.disable();
            }
        }, {
            immediate: true,
        });

        const check_actions = ref(null);
        const currentLocale = computed(() => usePage().props.value.locales[usePage().props.value.locale]);

        const PAYPAL_LOAD_SCRIPT_DATA = {
            "client-id": APP_MODE === 'local' && !isProductionMode ? PAYPAL_APP_CLIENT_ID : PAYPAL_APP_CLIENT_ID_LIVE,
            locale: currentLocale.regional,
            currency: props.form.currency,

            components: ['buttons', 'hosted-fields'],

            enableFunding: ['card', 'credit'],

            dataPageType: 'checkout',
            dataCspNonce: 'tmaltd'
        }

        // PayPal Subscription
        // Check if current page is subscription checkout
        const isSubscription = (props.subscription && usePage().props.value.url.current_route_name.includes('.subscription')) ? true : false;

        const getOrderDetails = () => {
            // Set payer's data
            const payer_info_data =  {
                name: {
                    given_name: props.form.first_name,
                    surname: props.form.last_name,
                },
                email_address: props.form.email,
            };

            // Set the payer's phone number, if provided
            if(props.form.phone) {
                payer_info_data['phone'] = {
                    phone_number: {national_number: props.form.phone}
                }
            }

            const orderConfig = {
                application_context: {
                    shipping_preference: 'NO_SHIPPING',
                    brand_name: usePage().props.value.options.CONFIG_SITE_NAME,
                },
            };

            if (isSubscription) {
                orderConfig['plan_id'] = props.subscription;
                orderConfig['subscriber'] = payer_info_data;

                if(orderID.value) {
                    orderConfig['custom_id'] = 'TMA-' + orderID.value;
                }

            } else {
                orderConfig['intent'] = 'CAPTURE';
                orderConfig['payer'] = payer_info_data;
                orderConfig['purchase_units'] = [{
                    amount: {
                        currency_code: props.form.currency,
                        value: Number(props.total).toFixed(2),
                        breakdown: {
                            item_total: {
                                currency_code: props.form.currency,
                                value: Number(Number(props.total) + Number(props.discount)).toFixed(2)
                            },
                            discount: {
                                currency_code: props.form.currency,
                                value: Number(props.discount).toFixed(2)
                            }
                        }
                    },
                    items: products.value,
                }];

                // orderConfig['payment_source'] = {
                //     [selectedPayment.value]: {

                //     }
                // }
            }

            return orderConfig;
        }

        const PAYPAL_BUTTONS_CONFIG = {
            onInit: function(data, actions) {
                PAY_PAL_ACTION.value = actions;
                loading.value = false;
            },
            onApprove: function(data, actions) {
                if (!isSubscription) {
                    // return actions.order.capture()
                    return axios.post(
                        _route(
                            'api.paypal.order.capture',
                            {
                                order_id: data.orderID
                            }
                        )
                    )
                    .then((response) => {
                        const orderData = response.data;
                        const errorDetail = Array.isArray(orderData.details) && orderData.details[0];

                        if (errorDetail && errorDetail.issue === 'INSTRUMENT_DECLINED') {
                            return actions.restart();
                        }

                        if (errorDetail) {
                            var msg = __('Sorry, your transaction could not be processed.');
                            if (errorDetail.description) msg += '\n\n' + errorDetail.description;
                            if (orderData.debug_id) msg += ' (' + orderData.debug_id + ')';
                            return alert(msg);
                        }

                        usePage().props.value.processing_order = true;
                        if(orderData.status == "COMPLETED") {
                            var transaction = orderData.purchase_units[0].payments.captures[0];

                            props.form
                            .transform((dataForm) => ({
                                ...dataForm,
                                payment: selectedPayment.value,
                                total: Number(props.total).toFixed(2),
                                order_status: 'completed',
                                transaction_id: transaction.id,
                            }))
                            .post(_route('shop.order.update', orderID.value));
                        }
                        else {
                            alert( __('Whoops! Something went wrong.') );
                            usePage().props.value.processing_order = false;
                        }
                    });
                } else {
                    return axios.post(
                        _route(
                            'api.paypal.subscription.details',
                            {
                                subscription_id: data.subscriptionID,
                            }
                        ), {
                            order_id: data.orderID
                        }
                    ).then((response) => {
                        const subscription = response.data

                        const custom_subscription_details = {
                            billing_info: subscription.billing_info,
                            id: subscription.id,
                            order_id: data.orderID,
                            plan_id: subscription.plan_id,
                            start_time: subscription.start_time,
                            subscriber: subscription.subscriber,
                            status: subscription.status,
                            status_changed_by: subscription.status_changed_by,
                            status_update_time: subscription.status_update_time,
                            create_time: subscription.create_time
                        }

                        props.form
                        .transform((dataForm) => ({
                            ...dataForm,
                            payment: selectedPayment.value,
                            total: Number(props.total).toFixed(2),
                            order_status: 'completed',
                            transaction_id: data.orderID,
                            is_subscription: isSubscription,
                            subscription: custom_subscription_details
                        }))
                        .post(_route('shop.order.update', orderID.value));
                    });
                }
            },
            onCancel: function (data, actions) {
                usePage().props.value.flash.error = __('Your transaction has been canceled!');
                usePage().props.value.processing_order = false;
            },
            onError: function (err) {
                usePage().props.value.flash.error = __('Error. Make sure that all required fields are filled in correctly.');
                usePage().props.value.processing_order = false;
            },
            onClick: function(e, actions)  {
                selectedPayment.value = e.fundingSource;
                check_actions.value = actions;

                if(!isServer && isValid.value){
                    usePage().props.value.hide_main_loading = false;
                    usePage().props.value.processing_order = true;
                    return axios.post(_route('shop.order.create'), {
                        ...props.form.data(),
                        payment: selectedPayment.value,
                        transaction_id: null,
                        total: Number(props.total).toFixed(2),

                        order_status: 'pending',
                        is_subscription: isSubscription
                    })
                    .then((response) => {
                        if(response.data.error) {
                            usePage().props.value.flash.error = response.data.error;
                            usePage().props.value.flash.message = null;
                            usePage().props.value.flash.success = null;

                            if(response.data.coupon == true){
                                usePage().props.value.auth.active_coupon = null;
                            }

                            usePage().props.value.processing_order = false;

                            if (response.data.action && response.data.action === 'refresh') {
                                Inertia.get(
                                    _route(
                                        usePage().props.value.url.current_route_name,
                                        {
                                            ...usePage().props.value.url.params,
                                            ...usePage().props.value.url.search_array
                                        }
                                    ),
                                    {},
                                    {
                                        preserveState: true,
                                        onSuccess: () => {
                                            usePage().props.value.flash.success = __('Cart amount has been updated');
                                        }
                                    }
                                )
                            }

                            return actions.reject();
                        }

                        if(!response.data.is_create){
                            console.log(__('Created order with status: ":status"', { status: __('custom.status.' + response.data.status) }));
                        }
                        else if(response.data.success) console.log(__(response.data.success));
                        orderID.value = response.data.order_id;

                        if (selectedPayment.value === 'card') {
                            usePage().props.value.hide_main_loading = true;
                        }

                        return actions.resolve();
                    })
                    .catch((error) => {
                        usePage().props.value.flash.error = error.response.data.message;
                        usePage().props.value.errors = error.response.data.errors;
                        usePage().props.value.processing_order = false;

                        return actions.reject();
                    });
                }
            },
            createSubscription: function(data, actions) {
                if(!isValid.value){
                    console.log('createSubscription: return false')
                    return false;
                }
                return actions.subscription.create(getOrderDetails())
            },
            createOrder: function(data, actions) {
                if(!isValid.value || isSubscription){
                    console.log('createOrder: return false')
                    return false;
                }

                const orderData = getOrderDetails()
                return axios.post(_route('api.paypal.order.create'), {
                    ...orderData,
                })
                .then((response) => {
                    return response.data.id
                })
                .catch(error => {
                    const err = error.response.data.error

                    if (err) {
                        err.details.forEach(el => {
                            if (el.field.includes('/phone_number')) {
                                usePage().props.value.errors['phone'] = __('validation.regex', {attribute: __('validation.attributes.phone')})
                            }

                            if (el.field.includes('/email')) {
                                usePage().props.value.errors['phone'] = __('validation.email', {attribute: __('validation.attributes.email')})
                            }

                            if (el.field.includes('/given_name')) {
                                usePage().props.value.errors['phone'] = __('validation.regex', {attribute: __('validation.attributes.first_name')})
                            }

                            if (el.field.includes('/surname')) {
                                usePage().props.value.errors['phone'] = __('validation.regex', {attribute: __('validation.attributes.last_name')})
                            }
                        })
                    }
                });
                // return actions.order.create(getOrderDetails());
            },
            style: {
                color:  'blue',
                shape:  'rect',
                label:  isSubscription ? 'subscribe' : 'pay',
                height: 50,
                size: 'responsive',
            }
        }

        if (isSubscription) {
            PAYPAL_LOAD_SCRIPT_DATA['vault'] = true;
            PAYPAL_LOAD_SCRIPT_DATA['intent'] = 'subscription';

            // delete PAYPAL_LOAD_SCRIPT_DATA['locale'];
            delete PAYPAL_LOAD_SCRIPT_DATA['currency'];

            delete PAYPAL_BUTTONS_CONFIG['createOrder']
        } else {
            // Init order function
            delete PAYPAL_BUTTONS_CONFIG['createSubscription']
        }

        const customCardFields = ref(null);

        // Init PayPal Buttons
        loadScript(PAYPAL_LOAD_SCRIPT_DATA)
        .then((paypal) => {
            paypal
                .Buttons(PAYPAL_BUTTONS_CONFIG)
                .render('#paypal-button-container')
                .catch((error) => {
                    console.error("Failed to render the PayPal Buttons", error);
                });
        })
        .catch((error) => {
            console.error("Failed to load the PayPal JS SDK script", error);
        });

        return {
            isValid,
            loading,

            products,

            customCardFields,

            hasRole
        }
    }
}
</script>

<style>

</style>
