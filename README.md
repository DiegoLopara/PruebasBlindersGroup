#Ejercicio 1
1.- Modificar la base de datos:
Agregar un nuevo campo en la tabla publicaciones (dbblog_post) para almacenar la fecha de publicación programada. 
public $scheduled_publish_date; //primero se declara la variable
'scheduled_publish_date' => array('type' => self::TYPE_DATE, 'validate' => 'isDateFormat', 'required' => false)
2.- Guardar la Fecha de Publicación Programada:
Modificar el método add() y update() en DbBlogPost:
public function add($autodate = true, $null_values = false)
{
    $this->scheduled_publish_date = Tools::getValue('scheduled_publish_date'); //recogemos el valor
    return parent::add($autodate, $null_values); //lo añadimos al parent
}

public function update($null_values = false)
{
    $this->scheduled_publish_date = Tools::getValue('scheduled_publish_date'); //Recogemos el valor
    return parent::update($null_values); //Lo actualizamos en el parent
}
3.- Filtrar Posts por Fecha:
Actualizar el método que recupera los posts:
Modificar el método que obtiene los posts para que solo devuelva aquellos cuya fecha de publicación programada ha llegado:
$sql = "SELECT * FROM "._DB_PREFIX_."dbblog_post WHERE active = 1 AND (scheduled_publish_date IS NULL OR scheduled_publish_date <= NOW())";
4.- Crear un Cron Job:
Crea un archivo cron.php en el módulo para actualizar el estado de los posts programados:
public static function publishScheduledPosts()
{
    // Obtener la fecha y hora actual
    $currentDateTime = date('Y-m-d H:i:s');

    // Consulta para seleccionar los posts programados que deben ser publicados
    $sql = "SELECT id_dbblog_post FROM " . _DB_PREFIX_ . "dbblog_post 
            WHERE scheduled_publish_date IS NOT NULL 
            AND scheduled_publish_date <= '$currentDateTime' 
            AND active = 0"; // Solo selecciona los que están inactivos

    $posts = Db::getInstance()->executeS($sql);

    if (!empty($posts)) {
        foreach ($posts as $post) {
            // Actualizar el estado del post a activo
            $updateSql = "UPDATE " . _DB_PREFIX_ . "dbblog_post 
                           SET active = 1, date_add = '$currentDateTime' 
                           WHERE id_dbblog_post = " . (int)$post['id_dbblog_post'];
            Db::getInstance()->execute($updateSql);
        }
    }
}




#Ejercicio 2
1.- Modificar la Base de Datos
Primero, se necesita cambiar la longitud de los campos firstname y lastname en la tabla ps_customer:
public static $definition = array(
    'table' => 'customer',
    'primary' => 'id_customer',
    'fields' => array(
        'firstname' => array('type' => self::TYPE_STRING, 'validate' => 'isName', 'size' => 512, 'required' => true),
        'lastname' => array('type' => self::TYPE_STRING, 'validate' => 'isName', 'size' => 512, 'required' => true)
    ),
);
2.- Modificar el formulario de registro
Si se desea que el formulario de registro también acepte nombres y apellidos más largos, debe asegurarse de que el HTML permita hasta 512 caracteres.
<input type="text" name="firstname" id="firstname" class="form-control" value="{$customer.firstname|escape:'html':'UTF-8'}" required="required" maxlength="512" />
<input type="text" name="lastname" id="lastname" class="form-control" value="{$customer.lastname|escape:'html':'UTF-8'}" required="required" maxlength="512" />
3.- Limpiar el caché para que los cambios tengan efecto.









#Ejercicio 3
1.- Modificar la base de datos:
CREATE TABLE `ps_product_related` (
    `id_product` INT(11) NOT NULL,
    `id_related` INT(11) NOT NULL,
    PRIMARY KEY (`id_product`, `id_related`),
    FOREIGN KEY (`id_product`) REFERENCES `ps_product`(`id_product`) ON DELETE CASCADE,
    FOREIGN KEY (`id_related`) REFERENCES `ps_product`(`id_product`) ON DELETE CASCADE
);
2.- Modificar el método que muestra el formulario de edición del producto:
Agregar un nuevo campo para seleccionar productos relacionados en la sección de configuración del producto.
public function hookDisplayAdminProductsExtra($params)
{
    $id_product = (int)$params['id_product'];
    
    // Obtener productos relacionados
    $related_products = Db::getInstance()->executeS("SELECT id_related FROM " . _DB_PREFIX_ . "product_related WHERE id_product = $id_product");
    
    // Obtener todos los productos para mostrar en el selector
    $products = Db::getInstance()->executeS("SELECT id_product, name FROM " . _DB_PREFIX_ . "product_lang WHERE id_lang = " . (int)$this->context->language->id);
    
    // Mostrar el formulario
    $this->context->smarty->assign([
        'related_products' => $related_products,
        'products' => $products,
    ]);
    
    return $this->display(__FILE__, 'views/templates/admin/admin_products_extra.tpl');
}
3.- Crear el archivo de plantilla admin_products_extra.tpl:
<div class="form-group">
    <label>{$this->l('Productos relacionados')}</label>
    <select name="related_products[]" multiple>
        {foreach from=$products item=product}
            <option value="{$product.id_product}" {if in_array($product.id_product, $related_products)}selected{/if}>{$product.name}</option>
        {/foreach}
    </select>
</div>
4.- Modificar el método processUpdateProduct para guardar la selección de productos relacionados en la base de datos:
public function processUpdateProduct($id_product)
{
    // Eliminar productos relacionados existentes
    Db::getInstance()->delete('product_related', 'id_product = ' . (int)$id_product);
    
    // Obtener productos relacionados seleccionados
    if (Tools::isSubmit('related_products')) {
        $related_products = Tools::getValue('related_products');
        foreach ($related_products as $id_related) {
            Db::getInstance()->insert('product_related', [
                'id_product' => (int)$id_product,
                'id_related' => (int)$id_related
            ]);
        }
    }
}
5.- Modificar la lógica de presentación en el frontend:
Modificar el método que obtiene productos relacionados seleccionados en lugar de solo los comprados juntos, si están disponibles.
public function getRelatedProducts($id_product)
{
    $related_products = Db::getInstance()->executeS("SELECT id_related FROM " . _DB_PREFIX_ . "product_related WHERE id_product = " . (int)$id_product);
    
    if (!empty($related_products)) {
        return array_map(function($item) {
            return $item['id_related'];
        }, $related_products);
    }
    
    // Si no hay productos relacionados, devolver lógica actual
    return $this->getProductsPurchasedTogether($id_product);
}


Todos los ejercicios han sido realizados utilizando código propio, basándome en funcionalidades parecidas encontradas en el repositorio de prestashop.

Un saludo

Diego López Araque




