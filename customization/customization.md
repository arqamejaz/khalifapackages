# Khalifa POS — Custom Modifications

This file documents every customization applied to this version. When a newer version is released, reference this file to re-apply all changes.

---

## 1. Bulk Price Update by Category

**What it does:** Adds an "Update Prices" button on the Products page that opens a modal. Select a category and enter a percentage — all variation prices in that category are updated by that percentage.

### Files Changed

#### `routes/web.php`
After `Route::resource('products', ProductController::class);`, add:
```php
Route::post('/pro', [ProductController::class, 'priceupdate']);
```

#### `app/Http/Controllers/ProductController.php`
Add this method before the final closing `}` of the class:
```php
public function priceupdate(Request $request)
{
    $category_to_update = request()->input('category_id');
    $percentage_to_update = request()->input('percentage');
    $products = DB::table('products')
        ->join('categories', 'products.category_id', '=', 'categories.id')
        ->join('variations', 'products.id', '=', 'variations.product_id')
        ->select('products.id', 'products.name as product_name', 'categories.name as category_name', 'variations.name as variation_name', 'variations.default_sell_price')
        ->where('categories.id', '=', $category_to_update)
        ->update([
            'sell_price_inc_tax' => DB::raw("sell_price_inc_tax * (1 + {$percentage_to_update} / 100)"),
            'default_sell_price' => DB::raw("default_sell_price * (1 + {$percentage_to_update} / 100)"),
        ]);
    return redirect('products');
}
```

#### `resources/views/product/index.blade.php`

**A) Add "Update Prices" button** inside `#product_list_tab` div, before the `@can('product.create')` block:
```blade
<button type="button" class="tw-dw-btn tw-bg-gradient-to-r tw-from-indigo-600 tw-to-blue-500 tw-font-bold tw-text-white tw-border-none tw-rounded-full pull-right tw-m-2" data-toggle="modal" data-target="#myModal">
    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
        fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
        stroke-linejoin="round" class="icon icon-tabler icons-tabler-outline icon-tabler-percentage">
        <path stroke="none" d="M0 0h24v24H0z" fill="none" />
        <path d="M17 17m-1 0a1 1 0 1 0 2 0a1 1 0 1 0 -2 0" />
        <path d="M7 7m-1 0a1 1 0 1 0 2 0a1 1 0 1 0 -2 0" />
        <path d="M6 18l12 -12" />
    </svg> Update Prices
</button>
```

Move the `<br><br>` that was inside `@can('product.create')` to AFTER the `@endcan`, so all three buttons align on the same row.

**B) Add modal** after `@include('product.partials.edit_product_location_modal')` and before `</section>`:
```blade
<!-- Price Update Modal -->
<div class="modal fade" id="myModal">
    <div class="modal-dialog">
        <div class="modal-content">
            <div class="modal-header">
                <h3 class="modal-title">Update Price by Percentage</h3>
            </div>
            <Form action="{{ url('/pro') }}" method="POST">
                @csrf
                <div class="modal-body">
                    <div class="col-md-4">
                        <div class="form-group">
                            {!! Form::label('category_id', __('product.category') . ':*') !!}
                            {!! Form::select('category_id', $categories, null, ['class' => 'form-control select2', 'required', 'style' => 'width:100%', 'id' => 'product_list_filter_category_id', 'placeholder' => __('lang_v1.all')]); !!}
                        </div>
                    </div>
                    <div class="col-sm-4">
                        <div class="form-group">
                            {!! Form::label('percentage', __('Percentage') . ':*') !!}
                            {!! Form::text('percentage', null, ['class' => 'form-control', 'required', 'placeholder' => __('Percentage')]); !!}
                        </div>
                    </div>
                </div>
                <div class="modal-footer">
                    <button type="Submit" class="btn btn-primary">Submit</button>
                    <button type="button" class="btn btn-danger" data-dismiss="modal">Close</button>
                </div>
            </Form>
        </div>
    </div>
</div>
<!-- End Price Update Modal -->
```

---

## 2. Custom Invoice Number in Sell POS

**What it does:** Adds an editable invoice number field on the POS create and edit screens. If left blank, the system auto-generates one. Gated behind the `edit_invoice_number` permission.

### Files Changed

#### `resources/views/sale_pos/partials/pos_form.blade.php`
After the `@endif` that closes the `show_invoice_scheme` block, add:
```blade
<div class="row">
    @can('edit_invoice_number')
    <div class="col-sm-4">
        <div class="form-group">
            {!! Form::label('invoice_no', __('sale.invoice_no') . ':') !!}
            {!! Form::text('invoice_no', null, ['class' => 'form-control', 'placeholder' => __('lang_v1.keep_blank_to_autogenerate')]); !!}
        </div>
    </div>
    @endcan
</div>
```

#### `resources/views/sale_pos/partials/pos_form_edit.blade.php`
After the line `<p><strong>@lang('sale.invoice_no'):</strong> {{$transaction->invoice_no}}</p>`, add:
```blade
<div class="row">
    @can('edit_invoice_number')
    <div class="col-sm-4">
        <div class="form-group">
            {!! Form::label('invoice_no', __('sale.invoice_no') . ':') !!}
            {!! Form::text('invoice_no', $transaction->invoice_no, ['class' => 'form-control', 'placeholder' => __('sale.invoice_no')]); !!}
        </div>
    </div>
    @endcan
</div>
```

#### `app/Http/Controllers/SellPosController.php`
Find the line:
```php
$invoice_no = $this->transactionUtil->getInvoiceNumber($business_id, 'final', $transaction->location_id);
```
Replace with:
```php
if (!isset($request->invoice_no) || empty($request->invoice_no)) {
    $invoice_no = $this->transactionUtil->getInvoiceNumber($business_id, 'final', $transaction->location_id);
} else {
    $invoice_no = $request->invoice_no;
}
```

---

## 3. Ledger Balance Fix (All Four Formats)

**What it does:** Fixes wrong balance shown in the Account Summary (top) of all four ledger formats. Without this fix, advance payments are excluded from the balance_due calculation, causing the top summary to show a higher balance than the running balance in the table below.

**Root cause:** The newer version split the paid calculation into `$total_paid` and `$total_transactions_paid`, then used the wrong one (`$total_transactions_paid`) for `$curr_due`, and also incorrectly subtracted `$total_advance_payment`.

### Files Changed

#### `app/Utils/TransactionUtil.php` — inside `getLedgerDetails()`

**A)** Find these three lines:
```php
$total_paid = $total_invoice_paid + $total_purchase_paid - $total_sell_return_paid - $total_purchase_return_paid + $total_excess_advance_payment - $total_advance_payment;

$total_transactions_paid = $total_invoice_paid + $total_purchase_paid - $total_sell_return_paid - $total_purchase_return_paid;

$curr_due = $total_invoice + $total_purchase - $total_transactions_paid + $beginning_balance + $opening_balance_due;
```

Replace with:
```php
$total_paid = $total_invoice_paid + $total_purchase_paid - $total_sell_return_paid - $total_purchase_return_paid + $total_excess_advance_payment;

$curr_due = $total_invoice + $total_purchase - $total_paid + $beginning_balance + $opening_balance_due;
```

**B)** Find the running balance loop (look for `$bal += ($credit - $debit);`). The block above it that zeroes out advance payments must be **commented out**:
```php
//NOTE:: Commented because of mismatch between final ledger table balance due and top balance due
// if (! empty($val['payment_method_key']) && $val['payment_method_key'] == 'advance') {
//     $credit = 0;
//     $debit = 0;
// }

$bal += ($credit - $debit);
```
> If this block is active (uncommented) in the new version, the running balance in the ledger table rows will exclude advance payments while the top summary includes them — causing a mismatch.

---

## Summary of All Changed Files

| File | Change |
|------|--------|
| `routes/web.php` | Add `POST /pro` route |
| `app/Http/Controllers/ProductController.php` | Add `priceupdate()` method |
| `resources/views/product/index.blade.php` | Add Update Prices button + modal |
| `resources/views/sale_pos/partials/pos_form.blade.php` | Add invoice_no input (create) |
| `resources/views/sale_pos/partials/pos_form_edit.blade.php` | Add invoice_no input (edit) |
| `app/Http/Controllers/SellPosController.php` | Conditional invoice_no logic |
| `app/Utils/TransactionUtil.php` | Fix ledger balance calculation |
